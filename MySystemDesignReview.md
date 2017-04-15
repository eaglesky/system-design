# Knowledge

## Network.

### How does webserver work with browser?

https://aprelium.com/data/doc/2/abyssws-linux-doc-html/howdowswork.html

### Web server(http server) vs application server.

http://www.javaworld.com/article/2077354/learn-java/app-server-web-server-what-s-the-difference.html
http://stackoverflow.com/questions/936197/what-is-the-difference-between-application-server-and-web-server/936257#936257
App server often includes web server. Web server is mainly responsible for receiving
requests and return static html contents, while app server has more complex business
logic that processes the request. Web servers provide the caching and scaling functionality
demanded by web access, while app servers provide application level services such
as connection pooling, object pooling, transaction support, messaging services etc.

### Web hosting service

A web hosting service is a type of Internet hosting service that allows individuals
and organizations to make their website accessible via the World Wide Web. Web hosts
are companies that provide space on a server owned or leased for use by clients, as
well as providing Internet connectivity, typically in a data center.
Example: Bluehost, Dreamhost, Go Daddy, Host Gator, pair Networks.

Different types of web hosting service
Shared, dedicated, VPS, Cloud.
https://hostingfacts.com/different-types-of-web-hosting/.

### Computer Protocols- TCP/IP, POP, SMTP, HTTP, FTP and More

http://vlaurie.com/computers2/Articles/protocol.htm

#### TCP vs UDP

https://www.howtogeek.com/190014/htg-explains-what-is-the-difference-between-tcp-and-udp/
UDP is less reliable but much faster, so it is preferable for transmitting time-critical
data, like live video stream and online game data.

#### Ports

Open tcp port 80(http) and tcp port 443(https) for the traffic coming into the LBs.
Opening tcp port 80 for the inside traffic after first layer of LB can save putting
SSL certificate on all of the web servers.
Open tcp port 3306(MySQL db server default)

### Cookies

Http Cookies and third-party cookies.
Drawbacks of cookies:https://en.wikipedia.org/wiki/HTTP_cookie#Drawbacks_of_cookies

### Sessions.

http://stackoverflow.com/questions/3804209/what-are-sessions-how-do-they-work
Three ways of managing sessions:
https://www.youtube.com/watch?v=32UGARg8AzU
Cookies
URL rewriting, Hidden fields
https://www.youtube.com/watch?v=xGAVFsLfn2w
https://www.youtube.com/watch?v=Pv3FST7OcvQ

## Load balacing

The DNS server can return the ip of load balancer instead of a server, and let the
load balancer decide which server the request should be sent to.

### Load balancing strategies

1.  Based on load of the server(how busy it is, the cpu usage, or the number of connections).
2.  Store different types of files on different servers, and use the suffix of url to discreminate them. 
3.  Round robin. Example: BIND(the most widely used Domain Name System (DNS) software on the Internet) uses this strategy to resolve a host name --- same host name, but different ip each time. The drawback of this strategy is that one or few server could always receive requests that require most computation resource, making them much busier than the rest. Another thing that can show it is a bad strategy is DNS caching, which exists on the user's computer(maintained by the OS and browser). This caching is a map of host name and ip address that will expire(clear) after a certain amount of time(TTL). It is only used for speeding up the response time  (https://www.lifewire.com/introduction-to-domain-name-system-817512). So if an user keeps doing expensive work, it will always be done on the same server during TTL and that server will be always busy. Therefore instead of relying on the DNS server to do the load balancing, it is better to let the DNS server return the ip of the load balancer.

Server-side sessions could cause problems for load balancing, because one session
for a user is only stored on one server. Solution: store all the sessions on a dedicated
session server, or the load balancer itself, which can be accessed by all servers.
However what if the server is down?
We could try using RAID to achieve data redundancy to some degree, but this is still
not a good solution, as it is possible that the whole server is down. A better way
to deal with it is to use multiple servers, which requires some syncing process.
Sticky session can also be a solution here, which can be implemented by storing an
id in the user's cookie and let LB to remember the mapping of that id to the ip of
the server the session belongs to.

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

## Caching.

### Cache could be in memory or on the disk, client side or server side.

Examples:

1. File based caching. Storeing a new file(html file) when new data is entered, like what is done by Craigslist. The upside of this approach is that the html file doesn't have to be regenerated everytime it's visited. The downside of it is due to the redundancy, it is hard to change the common style of the pages. And it could use too much space.

2. MySQL caching.

   query_cache_type = 1 is enough to enable it. Used for cache the query result.

3. Memcached

   Cache data and objects in the RAM. Use LRU to purge old data when it is full.

### Local cache and distributed cache.

#### Local cache

A cache could be local to an application instance and stored in-memory.
It is private and so different application instances could each have a copy of the
same cached data. This data could quickly become inconsistent between caches(because
after one write request sent to an application server, the server will update the
data and update/invalidate the corresponding one in the associated cache, making
it differ from the rest), so it may be necessary to expire data held in a private
cache and refresh it more frequently. In these scenarios it may be appropriate
to investigate the use of a shared or a distributed caching mechanism.

#### Distributed cache

A distributed cache may span multiple servers so that it can grow in size and in transactional capacity.

http://stackoverflow.com/questions/15457910/what-is-a-distributed-cache

#### Comparison of the two

https://dzone.com/articles/process-caching-vs-distributed

#### Caching patterns

* Cache-aside pattern:
  https://blog.cdemi.io/design-patterns-cache-aside-pattern/
  Example: Memcached + MySQL
  Memcached is just a cache, not a database!
* Cache-through pattern:
  Cache handles the requests from the web server and persisit the data to the backed databased. E.g. Redis.

#### Caching query result vs caching the objects(?)

http://www.lecloud.net/post/9246290032/scalability-for-dummies-part-3-cache

## Database.

### Database storage engine.

A database storage engine is the underlying software that a DBMS uses to create,read,
update and delete data from a database. The storage engine should be thought of as
a “bolt on” to the database (server daemon), which controls the database’s interaction
with memory and storage subsystems. Thus, the storage engine is not actually the database,
but a service that the database consumes for the storage and retrieval of information.
Given that the storage engine is responsible for managing the information stored
in the database, it greatly affects the overall performance of the database (or
lack thereof, if the wrong engine is chosen).
https://www.percona.com/blog/2016/01/06/mongodb-revs-you-up-what-storage-engine-is-right-part-1/
E.g., MyISAM, InnoDB, Memory, Archive, NDB..
MyISAM vs InnoDB(default of MySQL):http://blog.danyll.com/myisam-vs-innodb/
Comparison of all: http://zetcode.com/databases/mysqltutorial/storageengines/
Database engines are responsible for indexing, according to Wikipedia.

#### Database vs file system.

A database is generally used for storing related, structured data, with well defined
data formats, in an efficient manner for insert, update and/or retrieval (depending
on application).
On the other hand, a file system is a more unstructured data store for storing arbitrary,
probably unrelated data. The file system is more general, and databases are built
on top of the general data storage services provided by file system.

The file system is useful if you are looking for a particular file, as operating systems
maintain a sort of index. However, the contents of a txt file won't be indexed, which
is one of the main advantages of a database.

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
Typically database index is a datastructure like B+ tree(non-binary self-balancing
tree).  Note that B+ tree is different from B tree.

More details on indexing are explained in the book *Database System Concepts* by Abraham Silberschatz.

Apart from that, B+ tree is more cache friendly than Hashtable, when processing a range query or the consecutive search keys are closed to each other. But for point query, using hashtable residing in memory is extremely faster than using other datastructures. Like Redis, the time complexity is O(1+n/k) where n is the number of items and k the number of buckets (Redis ensures that n/k is small). When hashtable is stored on disk, the major overhead is reading the data into memory, and hashtable is not so much faster than B+ tree even for point query, which is why most of the RDMSs prefer B+ tree.

http://stackoverflow.com/questions/15216897/how-does-redis-claim-o1-time-for-key-lookup

### Database normalization and denormalization.

https://www.essentialsql.com/get-ready-to-learn-sql-database-normalization-explained-in-simple-english/
http://www.vertabelo.com/blog/technical-articles/denormalization-when-why-and-how

### ACID transaction.

* Atomicity. In a transaction involving two or more discrete pieces of information,
  either all of the pieces are committed or none are.
* Consistency. A transaction either creates a new and valid state of data, or, if
  any failure occurs, returns all data to its state before the transaction was started.
* Isolation. A transaction in process and not yet committed must remain isolated
  from any other transaction.
* Durability. Committed data is saved by the system such that, even in the event of
  a failure and system restart, the data is available in its correct state.

### Ways of improving database query performance.

* Optimizing the query itself, add caching.
* Indexing.
* Denormalization.

### DBMS types:

https://www.youtube.com/watch?v=jSXxrTNZwXA
Relational, Use SQL: MySQL
Object-oriented, Use OQL: ObjectDB.
NoSQL.

### Distributed databases(NoSQL).

#### CAP theorem

It is impossible for a distributed computer system to simultaneously provide more
than two out of three of the following guarantees:
Consistency, Availability, Partition Tolerance.
CAP is frequently misunderstood as if one had to choose to abandon one of the three
guarantees at all times. In fact, the choice is really between consistency and availability
for when a partition happens only; at all other times, no trade-off has to be made.
https://en.wikipedia.org/wiki/CAP_theorem
Strictly, most of the systems are not CAP-available if P happens. What this theorem
really says is that when P happens, you need to make a trade-off between C and A.
But you can never achieve 100% for both. E.g., if master-slaves is used for each shard,
and if read and write are performed on master, then strong consistency can be guaranteed,
but the level of availability is low when P happens between the router and the master.
We can change it by allowing read on slaves, which makes it no longer CAP-consistent
and increases the level of availability, but it is still not 100% available since
the write request could fail in such case.
See the section of "Many systems are neither linearizable nor CAP-available" in
https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html

* Eventual Consistency, Strong Eventual Consistency, Strong Consistency
  http://stackoverflow.com/questions/29381442/eventual-consistency-vs-strong-eventual-consistency-vs-strong-consistency
  Another good article on eventual consistency and strong consistency:
  https://cloud.google.com/datastore/docs/articles/balancing-strong-and-eventual-consistency-with-google-cloud-datastore/#h.w3kz4fze562t
  As I understand it, strong eventual consistency uses CRDT to avoid conficts and thus
  the data could be consistent after some time without resolving conficts before propagating
  the update.
  Tradeoff between consistency and latency:
  http://cs-www.cs.yale.edu/homes/dna/papers/abadi-pacelc.pdf
  Saved as CAP_tradeoff.pdf

#### Database partitioning.

https://en.wikipedia.org/wiki/Partition_(database)
Horizontal partitioning, or sharding in the distributed setting.
Vertical partitioning. Downside: need to do join when pulling data from multiple tables.
Upside is that some rarely-used columns are separated out, and when the machines that
have the columns are down, the impact is small.

* Consistent hashing:
  Sharding requires hashing the key and find the machine the data should be sent to.
  Simple hashing like key % n could lead to difficulties of moving the data when adding
  a new machine. To make it easier, consistent hashing is used instead.
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

#### Database replication

(1)Master-slave.
Master has a WAL(Write Ahead Log) to record every updates. Whenever an update happens,
it will notify all the slaves to read the log and apply the updates to their local
database.
As the master-slave replication is a one-way replication (from master to
slave),only the master database is used for the write operations, while read operations
may be spread on multiple slave databases(and the master too).
Upside: data replication due to the several backup slaves. Load balancing read to
the slaves so that it can process large amount of traffic and scale out easily.
Downside: 

1. Eventual consistency for reads to the slaves.(Might be acceptable)
2. Single-point-failure of the master db(short time, not able to receive any writes
   during that time), though it could be replaced by a promoted slave, the data could
   be lost if the newly written data hasn't been read, or the data on the slaves could
   be inconsistent if not all the slaves get the newly written data before the master
   is down. But this could be eventually resolved by some syncing process.
   https://blog.mlab.com/2013/03/replication-lag-the-facts-of-life/
3. Too many writes could go to the same master. May not be a big issue if one master
   only handles a shard. But still not the best choice when the traffic is write-heavy.、

(2)Peer to peer.
Upside: 
* If one master fails, other masters continue to update the database. Faster failover.
  Good for write-heavy traffic.
* Masters can be located in several physical sites, i.e. distributed across the network.
  Downside:
  Very hard to preserve absolute consistency(like write-write). 
  http://stackoverflow.com/questions/21196000/mysql-master-master-data-replication-consistency
  How to do the failover?? What does the consistency here mean? Cassandra example?

(3)Solutions to the eventual consistency:
1. Send critical read requests to the master so that they would always return the
   most up-to-date data;
2. Cache the data that has been written on the client side(or other places, for a
   period of time) so that you would not need to read the data you have just written.
3. Minimize the replication lag to reduce the chance of stale data being read
   from stale slaves.


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
NoSQL encompasses a wide variety of different database technologies that were developed
in response to the demands presented in building modern applications:
* Developers are working with applications that create massive volumes of new, rapidly
  changing data types — structured, semi-structured, unstructured and polymorphic data.
* Long gone is the twelve-to-eighteen month waterfall development cycle. Now small
  teams work in agile sprints, iterating quickly and pushing code every week or two,
  some even multiple times every day.
* Applications that once served a finite audience are now delivered as services that
  must be always-on, accessible from many different devices and scaled globally to
  millions of users.
* Organizations are now turning to scale-out architectures using open source software,
  commodity servers and cloud computing instead of large monolithic servers and storage
  infrastructure.
  Relational databases were not designed to cope with the scale and agility challenges
  that face modern applications, nor were they built to take advantage of the commodity
  storage and processing power available today.

##### Introduction of a few NoSQL databases.

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

##### Comparison of NoSQL and RDMS use cases.
* TBD


## Asynchronism

http://www.lecloud.net/post/9699762917/scalability-for-dummies-part-4-asynchronism

* Use cronjobs to convert dynamic content to static content
  (more info: https://vwo.com/knowledge/dynamic/).
  This is essentially a file-based caching and might be also done using document-based
  NoSQL databases with in-memory caching.
* Asynchronously process the expensive tasks.


# Interview strategies

## System analysis

* DAU: Daily Active User.
* MAU: Monthly Active User

 For games, DAU/MAU of ~20-30% is considered to be pretty good.
 For social apps, like a messenger app, a successful one would have a DAU/MAU closer to 50%. E.g. For Twitter, MAU = 320M, DAU = 150M(or 200M, 100M for the average)
 In general most apps struggle to get to DAU/MAU of 20% or more.

* Concurrent user number = DAU * session_seconds_per_user / 86400

 Peak_concurrent_user_number = concurrent_user_number * 3
 For Twitter, concurrent_user_number ~ 100K, peak ~ 300K

* For Twitter, Read QPS ~ 300K, write QPS ~ 6K.

 This might be estimated by doing:
 Read QPS ~ 150M * 60 / 86400 ~ 100K, 60 is estimated read per user.
 Write QPS ~ Read QPS / 60, 60 is estimated ratio of read to write.

* QPS vs Web/database server
  * QPS ~ 100: A laptop.
  * QPS ~ 1K: A web server/SQL database.
  * QPS ~ 10K: A NoSQL database like Cassandra.
  * QPS ~ (100K - 1M): A NoSQL database like like Memcached and Redis.

# Questions

1. In url shortening design, how to do sharding for the custom urls?