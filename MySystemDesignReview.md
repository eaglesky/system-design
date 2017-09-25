# Knowledge

## Network.

### How does webserver work with browser?

https://aprelium.com/data/doc/2/abyssws-linux-doc-html/howdowswork.html

### Web server(http server) vs application server.

http://www.javaworld.com/article/2077354/learn-java/app-server-web-server-what-s-the-difference.html
http://stackoverflow.com/questions/936197/what-is-the-difference-between-application-server-and-web-server/936257#936257
App server often includes web server. Web server is mainly responsible for receiving requests and return static html contents, while app server has more complex business logic that processes the request. Web servers provide the caching and scaling functionality demanded by web access, while app servers provide application level services such as connection pooling, object pooling, transaction support, messaging services etc.

### Web hosting service

A web hosting service is a type of Internet hosting service that allows individuals and organizations to make their website accessible via the World Wide Web. Web hosts are companies that provide space on a server owned or leased for use by clients, as well as providing Internet connectivity, typically in a data center.

Example: Bluehost, Dreamhost, Go Daddy, Host Gator, pair Networks.

Different types of web hosting service
Shared, dedicated, VPS, Cloud.
https://hostingfacts.com/different-types-of-web-hosting/.

### Computer Protocols- TCP/IP, POP, SMTP, HTTP, FTP and More

http://vlaurie.com/computers2/Articles/protocol.htm
FTP is application level, built on top of TCP. 

#### TCP vs UDP
When transmitting a large file, TCP could be slow if the distance is long. Because the large file transmitted package by package, whose size is determined by MTU. Each package will be only sent when the previous one is sent successfully to the destination and the sender got ACK. So the total transmission time = number of packages * RTT of each package. The latter is mostly affected by the distance.
http://moo.nac.uci.edu/~hjm/HOWTO_move_data.html

https://www.howtogeek.com/190014/htg-explains-what-is-the-difference-between-tcp-and-udp/
UDP is less reliable but much faster, so it is preferable for transmitting time-critical data, like live video stream and online game data.

#### Ports

Open tcp port 80(http) and tcp port 443(https) for the traffic coming into the LBs.
Opening tcp port 80 for the inside traffic after first layer of LB can save putting
SSL certificate on all of the web servers.
Open tcp port 3306(MySQL db server default)

### Cookies

Http Cookies and third-party cookies.
Drawbacks of cookies:https://en.wikipedia.org/wiki/HTTP_cookie#Drawbacks_of_cookies

### Sessions

http://stackoverflow.com/questions/3804209/what-are-sessions-how-do-they-work  
Three ways of managing sessions:  
https://www.youtube.com/watch?v=32UGARg8AzU  
Cookies  
URL rewriting, Hidden fields  
https://www.youtube.com/watch?v=xGAVFsLfn2w
https://www.youtube.com/watch?v=Pv3FST7OcvQ  
Sessions are useful to keep the user logged in. Usually a session table looks like:

| Column     | Type      | Meaning                         |
|:--------:  |:---------:|:-----------:                    |
|session_key | string    | A unique hash code              |
|user_id     |Foreign key|Point to the user table          |
|expire_at   |timestamp  |When to invalidate the session id|

The sessions can be stored in either the database or the cache. Preferably they should be centralized so that every web server could access it.

About sticky session:
https://stackoverflow.com/questions/1553645/pros-and-cons-of-sticky-session-session-affinity-load-blancing-strategy

One user can have multiple session_ids, each corresponds to a certain activity. This is probably why user_id is not used as session_id, otherwise user_id cannot be used as primary key for query. 
Also, by storing a session ID you can identify different sessions of the same user, and you may want to handle them in any special way (e.g. just allow a single session, or have data that's associated with the session instead of to the user).
And you can distinguish easly activity from different sessions, so you can kill a session without having to change your password if you left it open in a computer, and the other sessions won't notice a difference.
https://stackoverflow.com/questions/13146298/why-should-i-use-session-id-in-cookie-instead-of-storing-login-and-hashed-pass

## Load balacing

The DNS server can return the ip of load balancer instead of a server, and let the load balancer decide which server the request should be sent to. Usually a single load balancer can handle around 10M concurrent connections.
https://en.wikipedia.org/wiki/C10k_problem

### Load balancing strategies

1.  Based on load of the server(how busy it is, the cpu usage, or the number of connections).
2.  Store different types of files on different servers, and use the suffix of url to discreminate them. 
3.  Round robin. Example: BIND(the most widely used Domain Name System (DNS) software on the Internet) uses this strategy to resolve a host name --- same host name, but different ip each time. The drawback of this strategy is that one or few server could always receive requests that require most computation resource, making them much busier than the rest. Another thing that can show it is a bad strategy is DNS caching, which exists on the user's computer(maintained by the OS and browser). This caching is a map of host name and ip address that will expire(clear) after a certain amount of time(TTL). It is only used for speeding up the response time  (https://www.lifewire.com/introduction-to-domain-name-system-817512). So if an user keeps doing expensive work, it will always be done on the same server during TTL and that server will be always busy. Therefore instead of relying on the DNS server to do the load balancing, it is better to let the DNS server return the ip of the load balancer.

Server-side sessions could cause problems for load balancing, because one session for a user is only stored on one server. Solution: store all the sessions on a dedicated session server, or the load balancer itself, which can be accessed by all servers.

However what if the server is down?
We could try using RAID to achieve data redundancy to some degree, but this is still not a good solution, as it is possible that the whole server is down. A better way to deal with it is to use multiple servers(like Memcached), which requires some syncing process. Sticky session can also be a solution here, which can be implemented by storing an id in the user's cookie and let LB to remember the mapping of that id to the ip of the server the session belongs to. This is what NGINX load balancer can do: https://www.nginx.com/resources/glossary/load-balancing/. And the session data is always located in the server's memory. This can be used together with centralized session database in case one of the server that has the user's session is down.  
If the session data is not very large, it can also be stored in the user's browser cookie, or using URl rewriting. But this is not preferable since usually client should not be able to access the session data or has the chance of editing it. The session data should be only modified by the server(website).

Ref: https://shlomoswidler.com/2010/04/elastic-load-balancing-with-sticky-sessions.html

* RAID(Redundant array of inexpensive disks):
  It is a data storage virtualization technology that combines multiple physical disk drive components into a single logical unit for the purposes of data redundancy, performance improvement, or both.

  RAID0: Uses data striping(technique of segmenting logically sequential data, such as a file, so that consecutive segments are stored on different physical storage devices) to perform concurrent read and write, so that the performance could be improved by the number of disks. No data redundancy involved.

  RAID1: Uses data mirroring to implement redundancy. Typically two idendical hard drives. Write throughput is always slower because every drive must be updated, and the slowest drive limits the write performance. 

  RAID10: A combination of above.

### Load balancing replication

Multiple load balancers: active - active or active - passive.
One needs to send its heartbeats(packages) every period of time to another.
http://serverfault.com/questions/388543/load-balancing-active-active-config-and-dynamic-server-addition

### Load Balancer choices:

Software:
ELB(AWS Elastic Load Balancing), HAProxy, LVS(Linux Virtual Server, http://www.linuxvirtualserver.org/whatis.html)
Hardware:
Barracuda, Cisco, Citrix, F5..

## Caching

* Cache could be in memory or on the disk, client side or server side.

Examples:
1. File based caching. Storeing a new file(html file) when new data is entered, like what is done by Craigslist. The upside of this approach is that the html file doesn't have to be regenerated everytime it's visited. The downside of it is due to the redundancy, it is hard to change the common style of the pages. And it could use too much space.
2. MySQL caching.  
   query_cache_type = 1 is enough to enable it. Used for cache the query result.
3. Memcache, like Memcached.  
   Cache data and objects in the RAM. Use LRU to purge old data when it is full.

### Local cache and distributed cache.

#### Local cache
A cache could be local to an application server instance and stored in-memory.
It is private and so different application instances could each have a copy of the same cached data. This data could quickly become inconsistent between caches(because after one write request sent to an application server, the server will update the data and update/invalidate the corresponding one in the associated cache, making it differ from the rest), so it may be necessary to expire data held in a private cache and refresh it more frequently. In these scenarios it may be appropriate to investigate the use of a shared or a distributed caching mechanism.

#### Distributed cache
A distributed cache may span multiple servers so that it can grow in size and in transactional capacity. Different from local cache, it is shared by all application servers.  
http://stackoverflow.com/questions/15457910/what-is-a-distributed-cache  
Example: Memcached([Wikipedia](https://en.wikipedia.org/wiki/Memcached#Architecture)):  
>  If a client wishes to set or read the value corresponding to a certain key, the client's library first computes a hash of the key to determine which server to use. Then it contacts that server. This gives a simple form of sharding and scalable shared-nothing architecture across the servers. The server computes a second hash of the key to determine where to store or read the corresponding value.

#### Comparison of the two
https://dzone.com/articles/process-caching-vs-distributed

### Caching patterns

* Cache-aside pattern
  https://blog.cdemi.io/design-patterns-cache-aside-pattern/  
  Example: Memcached + MySQL
  (Memcached is just a cache, not a database!)  
  When getting a value from the cache, just use cache get, don't use containsKey first to check if key is in the cache -- it would result in an extra request to the cache, and even if it returns true, the value might be already purged when the second get request is sent out.  
  ``` java
      User getUser(long userId) {
        Key key = hash(userId);
        User user = cache.get(key);
        if (user != null) {
            return user;
        }
        user = database.get(userId);
        cache.set(key, user);
        return user;
      }

      void setUser(User user) {
        Key key = hash(user.userId);
        //Invalidate the data in cache first, not set, 
        //otherwise the data would be inconsistent if the next statement failed.
        //We might not need to do this if the set is to insert a new data,
        //since the new data has not been loaded in cache yet.
        cache.remove(key);
        //Need to make sure the cache remove succeed before proceed
        database.set(key);
      }
  ```
  

* Cache-through pattern
  Cache handles the requests from the web server and persisit the data to the backed databased. E.g. Redis.

  * CDN(Content Divery Network) uses this pattern. The cache servers are located around the world, all getting data from the original server. They are usually used for storing STATIC data, like images, videos and audios. Usually the webpage response contains the urls of them, which are resolved by the local DNS server to the CDN cache servers that are closest to the client. The cache server returned the stored data directly to the incoming request if it exists, otherwise it fetches the data from the original server, cache it and return it back. So essentially the cach server is a proxy, and it usually uses two ways of caching --- caching the data in memory and on the disk. I think in each location the CDN cache servers can be sharded too, if the data cannot fit in one server.  
  Reference: https://www.nczonline.net/blog/2011/11/29/how-content-delivery-networks-cdns-work/  
  This works great for transimtting media files because they are usually large and the latency is mostly affected by the physical distance between the server and clients. Getting the files is fast since it is just one file system read access. Similar optimization can be used if the distance between the sender and receiver plays a major role in latency -- like the queries sent to the cache from the webserver. Shortening the distance between the webserver and cache could greatly reduce the latency, so one way to do that would be to deploy webservers and the cache together and distributedly around the world. This works well if there are very few DB queries for each request, see more in [Tiny URL](tinyURL.md).

* Comparison of the two.
  Most applications leveraging global caches tend to use the first type, where the cache itself manages eviction and fetching data to prevent a flood of requests for the same data from the clients. However, there are some cases where the second implementation makes more sense. For example, if the cache is being used for very large files, a low cache hit percentage would cause the cache buffer to become overwhelmed with cache misses; in this situation, it helps to have a large percentage of the total data set (or hot data set) in the cache. Another example is an architecture where the files stored in the cache are static and shouldn’t be evicted. (This could be because of application requirements around that data latency—certain pieces of data might need to be very fast for large data sets—where the application logic understands the eviction strategy or hot spots better than the cache.)

### Caching query result vs caching the objects(?)
http://www.lecloud.net/post/9246290032/scalability-for-dummies-part-3-cache

### Cache usages
* Write-through cache directs write I/O onto cache and through to underlying permanent storage before confirming I/O completion to the host. This ensures data updates are safely stored on, for example, a shared storage array, but has the disadvantage that I/O still experiences latency based on writing to that storage. Write-through cache is good for applications that write and then re-read data frequently as data is stored in cache and results in low read latency. This way can be used for twitter timeline caching, but may allow some delay for writing to cache.
* Write-around cache is a similar technique to write-through cache, but write I/O is written directly to permanent storage, bypassing the cache. This can reduce the cache being flooded with write I/O that will not subsequently be re-read, but has the disadvantage is that a read request for recently written data will create a “cache miss” and have to be read from slower bulk storage and experience higher latency.
* Write-back cache is where write I/O is directed to cache and completion is immediately confirmed to the host. This results in low latency and high throughput for write-intensive applications, but there is data availability exposure risk because the only copy of the written data is in cache. As we will discuss later, suppliers have added resiliency with products that duplicate writes. Users need to consider whether write-back cache offers enough protection as data is exposed until it is staged to external storage. Write-back cache is the best performer for mixed workloads as both read and write I/O have similar response time levels.
* If there is some problem or it is expensive writing to cache(requires loading data from multiple sources), we can have a asyncynous server updating the cache periodically to make the data up-to-date. Need to maintain last update time(for each key). E.g. Facebook newsfeed.

## Database.

### Database storage engine.

A database storage engine is the underlying software that a DBMS uses to create,read, update and delete data from a database. The storage engine should be thought of as a “bolt on” to the database (server daemon), which controls the database’s interaction with memory and storage subsystems. Thus, the storage engine is not actually the database, but a service that the database consumes for the storage and retrieval of information.  
Given that the storage engine is responsible for managing the information stored in the database, it greatly affects the overall performance of the database (or lack thereof, if the wrong engine is chosen). https://www.percona.com/blog/2016/01/06/mongodb-revs-you-up-what-storage-engine-is-right-part-1/
E.g., MyISAM, InnoDB, Memory, Archive, NDB..  
MyISAM vs InnoDB(default of MySQL):http://blog.danyll.com/myisam-vs-innodb/  
Comparison of all: http://zetcode.com/databases/mysqltutorial/storageengines/
Database engines are responsible for indexing, according to Wikipedia.

#### Database vs file system.

A database is generally used for storing related, structured data, with well defined data formats, in an efficient manner for insert, update and/or retrieval (depending on application). On the other hand, a file system is a more unstructured data store for storing arbitrary, probably unrelated data. The file system is more general, and databases are built on top of the general data storage services provided by file system.

The file system is useful if you are looking for a particular file, as operating systems maintain a sort of index. However, the contents of a txt file won't be indexed, which is one of the main advantages of a database.

For very complex operations, the file system is likely to be very slow.

Main RDBMS advantages:
* Tables are related to each other
* SQL query/data processing language
* Transaction processing addition to SQL (Transact-SQL)
* Server-client implementation with server-side objects like stored procedures, functions,
  triggers, views, etc.

Advantage of the File System over Data base Management System is:
When handling small data sets with arbitrary, probably unrelated data, file is more efficient than database. For simple operations, read, write, file operations are faster and simple. Usually we prefer to store images and videos on the file system rather than database, since we don't really need to use those database features when fetching those files, which can be fetched efficiently enough using file system(path to disk location resolution is fast):

https://www.quora.com/Linux-Kernel-How-do-the-path-look-up-mechanism-namei-work-in-Linux

### Database indexing.

http://www.programmerinterview.com/index.php/database-sql/what-is-an-index/
Typically database index is a data structure like B+ tree(non-binary self-balancing tree).  Note that B+ tree is different from B tree.

More details on indexing are explained in the book *Database System Concepts* by Abraham Silberschatz. Typically they are large and stored on disks.

B+ trees are exceptionally good for range queries. Once the first record in the range has been found using the search algorithm described above, the rest of the records in the range can be found by sequential processing the remaining records in the leaf node, and then continuing down (actually right of the current leaf node) the linked list of leaf nodes as far as necessary. Once a record is found that has a search key value above the upper bound of the requested range, then the search completes.

Apart from that, B+ tree is more cache friendly than Hash table, when processing a range query or the consecutive search keys are closed to each other. But for point query, using hash table residing in memory is extremely faster than using other data structures. Like Redis, the time complexity is O(1+n/k) where n is the number of items and k the number of buckets (Redis ensures that n/k is small). When hash table is stored on disk, the major overhead is reading the data into memory, and hash table is not so much faster than B+ tree even for point query, which is why most of the RDMSs prefer B+ tree.

http://stackoverflow.com/questions/15216897/how-does-redis-claim-o1-time-for-key-lookup

A Primary index is an index whose search key also defines the sequential order of the file. The search key typically is the primary key, and is guaranteed not to contain duplicates.
A Secondary index is an index that is not a primary index and may have duplicates. eg. Employee name can be example of it. Because Employee name can have similar values.
https://www.quora.com/What-is-difference-between-primary-index-and-secondary-index-exactly-And-whats-advantage-of-one-over-another
https://stackoverflow.com/questions/20824686/what-is-difference-between-primary-index-and-secondary-index-exactly
  * Global and local secondary index. http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html

### Normalization and denormalization.
Normalize until it hurts, denormalize until it works.  
Build your database tier using everything they taught you in the classroom. Normalization is your friend here. Why? When you've effectively normalized your database, you can monitor the results of queries and eventually introduce the optimizations that you need to make things run faster.

https://www.essentialsql.com/get-ready-to-learn-sql-database-normalization-explained-in-simple-english/
http://www.vertabelo.com/blog/technical-articles/denormalization-when-why-and-how

### Schema design cases
#### Many to Many relation
* http://www.joinfu.com/2005/12/managing-many-to-many-relationships-in-mysql-part-1/
* https://stackoverflow.com/questions/3653462/is-storing-a-delimited-list-in-a-database-column-really-that-bad/3653574#3653574

##### Friendship table
* One direction friendship is easy to store. 
* Two directions friendship can be represented by two records or one record.
  * If A and B become friends, store A -> B and B -> A. The query for finding the friends of A only needs to check one column. But need to write twice when saving a new friendship. And there could be consistency issue if one of the write fails. We can solve this by using a transaction, or have a cron job to correct the inconsistencies periodically. This is usually more performant than the second approach, either in SQL or NoSQL.
  * If A and B become friends, store A -> B only. The query for finding the friends of A needs to check both columns.
* Usually two ids as primary keys are enough. Typically the first id is used as shard key if the table is distributed. This won't cause serious unbalancing issue, and if it does, deal with it, like Facebook TAO.


### ACID transaction.

* Atomicity. In a transaction involving two or more discrete pieces of information, either all of the pieces are committed or none are.
* Consistency. A transaction either creates a new and valid state of data, or, if any failure occurs, returns all data to its state before the transaction was started.
* Isolation. A transaction in process and not yet committed must remain isolated from any other transaction.
* Durability. Committed data is saved by the system such that, even in the event of a failure and system restart, the data is available in its correct state.

### Improving database query performance.

* Optimizing the query itself, add caching.
* Indexing.
* Denormalization.

### DBMS types

https://www.youtube.com/watch?v=jSXxrTNZwXA  
* Relational, Use SQL: MySQL  
* Object-oriented, Use OQL: ObjectDB.  
* NoSQL.

### Distributed databases(NoSQL).

#### CAP theorem

It is impossible for a distributed computer system to simultaneously provide more than two out of three of the following guarantees:  
Consistency, Availability, Partition Tolerance.
CAP is frequently misunderstood as if one had to choose to abandon one of the three guarantees at all times. In fact, the choice is really between consistency and availability for when a partition happens only; at all other times, no trade-off has to be made.  
https://en.wikipedia.org/wiki/CAP_theorem
Strictly, most of the systems are not CAP-available if P happens. What this theorem really says is that when P happens, you need to make a trade-off between C and A. But you can never achieve 100% for both. E.g., if master-slaves is used for each shard, and if read and write are performed on master, then strong consistency can be guaranteed, but the level of availability is low when P happens between the router and the master.
We can change it by allowing read on slaves, which makes it no longer CAP-consistent and increases the level of availability, but it is still not 100% available since the write request could fail in such case.
See the section of "Many systems are neither linearizable nor CAP-available" in
https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html

* Eventual Consistency, Strong Eventual Consistency, Strong Consistency  
  http://stackoverflow.com/questions/29381442/eventual-consistency-vs-strong-eventual-consistency-vs-strong-consistency
  Another good article on eventual consistency and strong consistency:  
  https://cloud.google.com/datastore/docs/articles/balancing-strong-and-eventual-consistency-with-google-cloud-datastore/#h.w3kz4fze562t  
  As I understand it, strong eventual consistency uses CRDT to avoid conficts and thus the data could be consistent after some time without resolving conficts before propagating the update.  
  Tradeoff between consistency and latency:  
  http://cs-www.cs.yale.edu/homes/dna/papers/abadi-pacelc.pdf  
  Saved as CAP_tradeoff.pdf

#### Database partitioning

* https://en.wikipedia.org/wiki/Partition_(database)  
  Horizontal partitioning, or sharding in the distributed setting.  
  Vertical partitioning. Downside: need to do join when pulling data from multiple tables, and when a table is too large, still need to use horizontal partitioning. Upside is that some rarely-used columns are separated out, and when the machines that have the columns are down, the impact is small.

* Consistent hashing:
  Sharding requires hashing the key and find the machine the data should be sent to. Simple hashing like key % n could lead to difficulties of moving the data when adding a new machine. To make it easier, consistent hashing is used instead.
  * Consistent hashing I:
    Implementation: http://blog.csdn.net/jmspan/article/details/51740066
    (How to deal with deleting a machine?)  
    Problem: 
   1. Too many reads in a short time on 1 ~ 2 machines.
   2. The data is not evenly distributed.
  
  * Consistent hashing II(solve the above problems):  
    Algorithm:
    http://afghl.github.io/2016/07/04/consistent-hashing.html
    https://loveforprogramming.quora.com/Distributed-Systems-Part-1-A-peek-into-consistent-hashing  
    Implementation:
    http://blog.csdn.net/jmspan/article/details/51749521

  The map of hashcodes to server id(routing table) is usually stored in the web server or config server.  
  Data on each virtual node can be replicated to the two other servers represented by the following virtual nodes(clockwise).  
  https://www.jiuzhang.com/qa/2280/  
  https://www.jiuzhang.com/qa/980/  
  Good thing about this is that when suddenly one server is down, the request for the data on that server will go to the server represented by the next virtual node(assuming the routing table on is updated in time upon failure) and thus get the replica data. Even if that server is down too, we still have replica data on the third server so the system can still be available(that's why we have two more copies on the next servers clockwise).

* Usually there is a router to direct the query to the right shard(s), and merge the results if necessary and return to the client. There could be another config server storing meta data about chunks on the shards, and range of each shard.  
E.g. https://docs.mongodb.com/manual/core/sharded-cluster-query-router/#routing-and-results-process
https://docs.mongodb.com/manual/core/sharded-cluster-config-servers/#config-servers

* The choice of shard key is very important. Sometimes if there could be uneven requests using user_id, then choosing user_id as shard key may cause some servers get too much load(hotspots). The solutions I've found are:
  1. Choose another shard key, like id.
  2. Still use user_id, but replicating the data to more machines.
  3. Add cache servers to cache the hot data. Cache servers can handle more QPS. But may still need to use method 2 to mitigate hot spot issue.
  4. Identify hot spot through proper monitoring and manually split the data to other servers.


#### Database replication
* In addition to increase availablility, database replication can also help distribute incoming traffic.
* Master-slave.
  * Master has a WAL(Write Ahead Log) to record every updates. Whenever an update happens, it will notify all the slaves to read the log and apply the updates to their local database.  
  As the master-slave replication is a one-way replication (from master to slave),only the master database is used for the write operations, while read operations may be spread on multiple slave databases(and the master too).
  * Upside: data replication due to the several backup slaves. Load balancing read to the slaves so that it can process large amount of traffic and scale out easily.
  * Downside: 
    - Eventual consistency for reads to the slaves. Might be acceptable. Can achieve strong consistency if reads and writes all go to the master.
    - Single-point-failure of the master db(short time, not able to receive any writes during that time), though it could be replaced by a promoted slave, the data could be lost if the newly written data hasn't been read, or the data on the slaves could be inconsistent if not all the slaves get the newly written data before the master is down. But this could be eventually resolved by some syncing process.  
    https://blog.mlab.com/2013/03/replication-lag-the-facts-of-life/
    - Too many writes could go to the same master. May not be a big issue if one master only handles a shard. But still not the best choice when the traffic is write-heavy.
  * When used with sharding, usually each shard consists of a master-slave cluster. E.g. MongoDB.

* Peer to peer.
  * Upside: 
    - If one master fails, other masters continue to update the database. Faster failover. Good for write-heavy traffic.
    - Masters can be located in several physical sites, i.e. distributed across the network.
  * Downside:
    - Very hard to preserve absolute(strong) consistency(like write-write). If we wait until the syncing process finishes, it will take too long and may be unacceptable. http://stackoverflow.com/questions/21196000/mysql-master-master-data-replication-consistency  
    How to do the failover?? What does the consistency here mean? Cassandra example?
  * When used with sharding, usually each data is replicated to two/three other machines. See the consistent hashing section.

* Solutions to the eventual consistency:
  1. Send critical read requests to the master so that they would always return the most up-to-date data;
  2. Cache the data that has been written on the client side(or other places, for a period of time) so that you would not need to read the data you have just written.
  3. Minimize the replication lag to reduce the chance of stale data being read from stale slaves.


#### NoSQL databases.

##### General introduction.

Good introduction video: https://www.youtube.com/watch?v=qI_g07C_Q5I

Hard to define, usually has characteristics:
Non-relational, cluster-friendly, schema-less, open-source.

Types:
Key-value: Redis(CP?), Memcached, Oracle NoSQL database.
Document: MongoDB(CP, by default read and write go to the master)
Column: HBase(CP), Cassandra(AP)
Graph: Neo4j(CP)

Why NoSQL?
NoSQL encompasses a wide variety of different database technologies that were developed in response to the demands presented in building modern applications:
* Developers are working with applications that create massive volumes of new, rapidly changing data types — structured, semi-structured, unstructured and polymorphic data.
* Long gone is the twelve-to-eighteen month waterfall development cycle. Now small teams work in agile sprints, iterating quickly and pushing code every week or two, some even multiple times every day.
* Applications that once served a finite audience are now delivered as services that must be always-on, accessible from many different devices and scaled globally to millions of users.
* Organizations are now turning to scale-out architectures using open source software, commodity servers and cloud computing instead of large monolithic servers and storage infrastructure. Relational databases were not designed to cope with the scale and agility challenges that face modern applications, nor were they built to take advantage of the commodity storage and processing power available today.

##### Introduction to a few NoSQL databases.

* Redis
  Introduction and Data Model: https://redis.io/topics/introduction
  Persistence: https://redis.io/topics/persistence
  		 http://oldblog.antirez.com/post/redis-persistence-demystified.html
  Cluster and sharding: https://redis.io/topics/partitioning
  	 https://redis.io/topics/cluster-tutorial
  vs Memcached, and why use Redis:
  http://stackoverflow.com/questions/7888880/what-is-redis-and-what-do-i-use-it-for

* MongoDB
  Introduction and Data Model: https://www.youtube.com/watch?v=pWbMrx5rVBE
  Sharding: https://docs.mongodb.com/manual/sharding/
  Architecture: https://www.mongodb.com/presentations/webinar-everything-you-need-know-about-sharding?jmp=docs&_ga=1.78388884.1635526347.1487659117

* HBase
  Introduction and Data Model: http://hbase.apache.org/0.94/book/datamodel.html
  Data Model and Architecture:  https://www.edureka.co/blog/hbase-architecture/

* Cassandra
  * Data format: `<row_key, column_key, value>`. 
   * row_key is the hash key and can not do range query on it. Like user_id.
   * column_key is sorted(either by time or name) and can do range query. Can be compound value like timestamp + user_id.
   * value is usally a string, or other object.
   * Example: Use `<owner_id, created_at, tweet_id>` to represent an owner with owner_id posted a tweet with tweet_id at created_at. Storing the data in this way can make it easy to solve questions like given a user, find out all his tweets between two times.
   * Another example: `<user_id, friend_user_id, <is_mutual_friend, is_blocked, timestamp>>`
  * More:
    * Paper: *Cassandra - A Decentralized Structured Storage System*
    * https://www.tutorialspoint.com/cassandra/cassandra_data_model.htm

##### Comparison of NoSQL and SQL.
* Many NoSQL databases do not(or cannot easily) provide good ACID transaction support.  
  https://dzone.com/articles/workaround-lack-full-atomic
* For SOME NoSQL databases, you have to handle data serialization and secondary index yourself?
* NoSQL can handle much higher QPS than SQL(single box). But SQL can still achieve good performance with enough number of servers.
* NoSQL has handled sharding and replica already.
* More? https://docs.microsoft.com/en-us/azure/documentdb/documentdb-nosql-vs-sql


## Asynchronism

http://www.lecloud.net/post/9699762917/scalability-for-dummies-part-4-asynchronism

* Use cronjobs to convert dynamic content to static content
  (more info: https://vwo.com/knowledge/dynamic/).
  This is essentially a file-based caching and might be also done using document-based
  NoSQL databases with in-memory caching.
* Asynchronously process the expensive tasks.

## System analysis
### DAU and MAU
* DAU: Daily Active User.
* MAU: Monthly Active User
For games, DAU/MAU of ~20-30% is considered to be pretty good.
For social apps, like a messenger app, a successful one would have a DAU/MAU closer to 50%. E.g. For Twitter, MAU = 320M, DAU = 150M(or 200M, 100M for the average). For Facebook, MAU = 1.59 Billion, DAU = 1 Billion.
In general most apps struggle to get to DAU/MAU of 20% or more.

* Concurrent user number = DAU * session_seconds_per_user / 86400
Peak_concurrent_user_number = concurrent_user_number * 3
For Twitter, concurrent_user_number ~ 100K, peak ~ 300K

* For Twitter, Read QPS ~ 300K, write QPS ~ 6K.
This might be estimated by doing:
Read QPS ~ 150M * 60 / 86400 ~ 100K, 60 is estimated read per user.
Write QPS ~ Read QPS / 60, 60 is estimated ratio of read to write.

### Some System Concepts
* **Latency** is the delay incurred in communicating a message (the time the message spends “on the wire”). 
* **Processing time** is the amount of time a system takes to process a given request, not including the time it takes the message to get from the user to the system or the time it takes to get from the system back to the user.
* **Response Time** is the total time it takes from when a user makes a request until they receive a response. Latency + Processing Time = Response Time. Refer to http://www.javidjamae.com/2005/04/07/response-time-vs-latency/ for more of above.
* Throughput:  For Application Server, throughput can be defined as the number of requests **processed** per minute per server instance, which is a function of many factors, including the nature and size of user requests, number of users, and performance of Application Server instances and back-end databases. For HADB, throughput can be defined as volume of session data stored per minute, which is the product of the number of HADB requests per minute, and the average session size per request.  
https://docs.oracle.com/cd/E19900-01/819-4741/abfcd/index.html

* QPS vs Web/database server
  * QPS ~ 100: A laptop.
  * QPS ~ 1K: A web server/SQL database.
  * QPS ~ 10K: A NoSQL database like Cassandra.
  * QPS ~ (100K - 1M): A NoSQL database like like Memcached and Redis.  
https://academy.datastax.com/planet-cassandra/nosql-performance-benchmarks
The above are just estimation for common uses. Actual throughput depends on the hardware usage, and also the processing time of each request/query.

### Analysis Steps
* Estimate scale, starting from DAU. (e.g., number of new tweets, number of tweet views, how many timeline generations per sec., etc.).
* Estimate storage. 
* Estimate ingress and egress network bandwidth. And QPS.

## System Design Workflow
* Scnario. 
  - Use cases.
  - Esimation. Usually need to estimate DAU and QPS. For storage, it maybe okay to delay estimating the number.
* Service and Storage. 
  - Try to give a high level design first that handles all the required use cases. After this, we should be clear about what services and tables are involved. 
  - Note that each service usually correspond to a servlet that handles particular requests. Difference services can run either on the same server or different servers. As long as they don't store data(stateless), they will perform the same, and scale horizontally in the same way. For each request, total_processing_time = in_queue_time + actual_processing_time. Since different services typically don't share any data, when they run in parallel on the same machine, they should not affect each other, and thus actual_processing_time are the same. in_queue_time should be the same too, so total_processing_time should be equal.
  - Dive into each part, figuring out how to store each table, and the workflow of handling the requests.
* Scale.

# Design Examples
* Twitter -- timeline and newsfeed. (See Nine Chapter slide for details)
* [Web crawler](crawler.md)
* [Type-ahead/Google Suggestion](typeahead.md)
* [Distributed file system](fileSystemDesign.md)
* [Tiny URL](tinyURL.md). Application of internationally distributed DB, compound id that contains shard key, global unique ID for distributed DB.


# Small design cases
* How to find mutual friends?  
  http://www.jiuzhang.com/qa/954/  
  I personally think using a graph database is a good idea!
  How Facebook store friends?
  https://www.jiuzhang.com/qa/4867/
  https://www.facebook.com/notes/facebook-engineering/tao-the-power-of-the-graph/10151525983993920
* How to implement pagination?  
  http://www.jiuzhang.com/qa/1839/
* How to get media info for each post?
* In url shortening design, how to do sharding for the custom urls?