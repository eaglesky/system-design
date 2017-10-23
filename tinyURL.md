# TinyURL 

## Scenario 
### Use cases
* Shortening: Take a url and return a much shorter url. 
	- Ex: http://www.interviewbit.com/courses/programming/topics/time-complexity/ => http://goo.gl/GUKA8w/
	- Gotcha: What if two people try to shorten the same URL?
* Redirection: Take a short url and redirect to the original url. 
	- Ex: http://goo.gl/GUKA8w => http://www.interviewbit.com/courses/programming/topics/time-complexity/
* Custom url: Allow the users to pick custom shortened url. 
	- Ex: http://www.interviewbit.com/courses/programming/topics/time-complexity/ => http://goo.gl/ib-time 
* Analytics: Usage statistics for site owner. 
	- Ex: How many people clicked the shortened url in the last day? 
* Automatic link expiration
* Manual link removal
* UI vs API

### Design goals 
* Latency
	- Our system is similar to DNS resolution, higher latency on URL shortener is as good as a failure to resolve.
* Consistency vs Availability
	-  Both are extremenly important. However, CAP theorem dictates that we choose one. Do we want a system that always answers correctly but is not available sometimes? Or else, do we want a system which is always available but can sometime say that a URL does not exists even if it does? This tradeoff is a product decision around what we are trying to optimize. Let's say, we go with consistency here.
* URL as short as possible
	- URL shortener by definition needs to be as short as possible. Shorter the shortened URL, better it compares to competition.

### Estimation 
* DAU = 100M.
* QPS.
  - Say each user wants to shorten 1 URL per day. Write QPS = 100M / 86400 ~ 1K. Peak write QPS ~ 2K
  - Say each generated url is clicked 10 times per day. Since there could be 100M new urls generated per day, after 1 year, there could be 100M x 365 = 3 x 10^10 urls. The read QPS could be 3 x 10^10 x 10 / (8 x 10^4) = 4M. 
* Storage.
  - Say on average the total size of original and generated url is 200. After one year, it takes 3 x 10^10 x 200 = 6TB.

## Service 
```java
class TinyURL
{
	map<longURL, shortURL> longToShortMap;
	map<shortURL, longURL> shortToLongMap;
	
	shortURL insert( longURL )
	{
		if longToShortMap not containsKey longURL
			generate shortURL;
			put<longURL, shortURL> into longToShortMap;
			put<shortURL, longURL> into shortToLongMap;
		return longToShortMap.get(longURL);
	}

	longURL lookup( shortURL )
	{
		return shortToLongMap.get( shortURL );
	}
}
```

### shortURL insert( longURL )
#### Encode 
##### Traditional hash function
* Types
	- Crypto hash function: MD5 and SHA-1
		+ Secure but slow
	- Fast hash function: Murmur and Jenkins
		+ Performance
		+ Have 32-, 64-, and 128-bit variants available

* Pros
	- No need to write additional hash function, easy to implement
	- Are randomly distributed
	- Support URL clean

* Cons

| Problem                             | Possible solution                                                                                                                                              | 
|-------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------| 
| Not short enough (At least 4 bytes) | Only retrieve first 4-5 digit of the hash result                                                                                                               | 
| Collision cannot be avoided         | Use long_url + timestamp as hash argument, if conflict happens, try again (timestamp changes) -> multiple attempts, highly possible conflicts when data is big | 
| Slow                                |                                                                                                                                                                | 

##### Base10 / Base62
* Base is important

| Encoding           | Base10     | Base62      | 
|--------------------|------------|-------------| 
| Year               | 36,500,000 | 36,500,000  | 
| Usable characters  | [0-9]      | [0-9a-zA-Z] | 
| Encoding length    | 8          | 5           | 

* Pros:
	- Shorter URL
	- No collision
	- Simple computation

* Cons:
	- No support for URL clean

#### Long to short with Base62 
```java
    public String longToShort( String url ) 
    {
        if ( url2id.containsKey( url ) ) 
        {
            return "http://tiny.url/" + idToShortKey( url2id.get( url ) );
        }
        GLOBAL_ID++;
        url2id.put( url, GLOBAL_ID );
        id2url.put( GLOBAL_ID, url );
        return "http://tiny.url/" + idToShortKey( GLOBAL_ID );
    }

    private String idToShortKey( int id )
    {    	
        String chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
        String short_url = "";
        while ( id > 0 ) 
        {
            short_url = chars.charAt( id % 62 ) + short_url;
            id = id / 62;
        }
        while ( short_url.length() < 6 ) 
        {
            short_url = "0" + short_url;
        }
        return short_url;
    }
```

### longURL lookup( shortURL ) 
#### Short to long with Base62
```java
    public String shortToLong( String url ) 
    {
        String short_key = getShortKey( url );
        int id = shortKeytoID( short_key );
        return id2url.get( id );
    }

    private String getShortKey( String url ) 
    {
        return url.substring( "http://tiny.url/".length() );
    }

    private int shortKeytoID( String short_key ) 
    {
        int id = 0;
        for ( int i = 0; i < short_key.length(); ++i ) 
        {
            id = id * 62 + toBase62( short_key.charAt( i ) );
        }
        return id;
    }
```

## Storage 
### SQL 
#### Schema design 
* Two maps
	- longURL -> shortURL
	- shortURL -> longURL
* In order to store less data ( Given shortURL, its corresponding sequential ID can be calculated )
	- longURL -> Sequential ID
	- Sequential ID -> longURL
* Create index on longURL column, only needs to store one table
	- Sequential ID -> longURL

### NoSQL 

## Scale 
### How to reduce response time?
#### Cache
#### Optimize based on geographical info
* Web server
	- Different web servers deployed in different geographical locations
	- Use DNS to parse different web servers to different geographical locations
      - Try make the distance between user to server shorter. Usually the communication between servers are way faster than the one between an end user and a server. 
* Database 
	- Centralized MySQL + Distributed memcached server
	- Web and cache servers are deployed in different geographical locations
        * This achitecture is good when there is only one or very few DB request for each user request sent to the web server. When there are many DB request for each user request, the distance between DB and web server becomes more important and can greatly affect the DB request latency. In that case, it is better to put DB closer to the web servers, and preferable the web servers should be in the same data center as the DB, instead of being distributed globaly.
        * The data in the cache should be consistent with the one in the DB. For tiny URL, the data are not modified, so it is safe to use local cache.
        
### What if one MySQL server could not handle
#### Problematic scenarios
* Too many write operations
* Too many information to store on a single MySQL database
* More requests could not be resolved in the cache layer

#### Sharding with multiple MySQL instances
* Vertical sharing
	- Only one table
	- Even with Custom URL, two tables in total
* Horizontal sharding: Choose sharding key?
	- Use Long Url as sharding key
		+ Short to long operation will require lots of cross-shard joins
	- Use ID as sharding key
    + Need to choose a proper way to get gloabl uniqu id, see next section for how. 
		+ Short to long url: First convert shourt url to ID; Find database according to ID; Find long url in the corresponding database
		+ Long to short url: Broadcast to N databases to see whether the link exist before. If not, get the next ID and insert into database. An alterative is to have another table(or cache all/recent) storing long_url -> id. Write should insert new data into two tables(write around or write through). Write is more costly and space usuage is higher, which might be acceptable.
	- Combine short Url and long Url together, use long url part as sharding key.
		+ ShortUrl = Hash(longUrl) % 62 + shortkey
    + Given longURL, get the shard id using consistent_hash(Hash(longURL) % 62). Check if there is already one row with longUrl equal to the given one(assuming longUrl is indexed on that machine). If not, then store <url_id, longUrl> into the table, where url_id is the concatentation of hash_number mod by 62 and sequencial_id_assigned_by_that_machine. It could be either a number or a string(the latter is easier to implement and can be indexed on that machine). One table is enough.
		+ Given shortURL, convert it into url_id, we can get the sharding machine from the first part of it.
		+ This way can be applied to similar cases when we want to query a distributed database by two different keys. If it is okay to shard by key A(no hot spot issue), and key B is just an id and does not represent any meaningful thing, we can construct B so that it is a concatenation of key A(or its hashcode) and sequential id on that machine.
	- Sharding according to the geographical info. This can be used together with the previous approach. 
		+ First find out which websites are more popular in which region. Put all websites popular in US in US DB. Given a long url, the shard id can be determined by first get the country that the url belongs to, then use consistent hashing to determine the exact shard id within that DC. E.g. long url = "http://www.sina.com/asdfsadf/sdfas", could be shortened to "http://myurl/04AB12345", 0 corresponds to China, and 4 = hash(longUrl) % 62. 


#### How to get global unique ID?
* We only need global unique ID here, and it doesn't have to be strictly sequential, which is very hard to implement in distributed systems: https://stackoverflow.com/questions/2671858/distributed-sequence-number-generation/5685869. 
* There are several ways of implementing global unique ID: http://srinathsview.blogspot.com/2012/04/generating-distributed-sequence-number.html
    - Zookeeper(might be a bit slow). On start-up there are bunch of ranges allocated and shared by all workers(like [1..10^6 ],[ 10^6+1, 2x10^6 ],...,[999x10^9, 10^12 ]). Each worker will randomly pick a unused range from it and then mark it 'in use', and then will keep returning an increasing number from that range for incoming requests. After all the numbers in that range are exausted, the worker will pick a new unused range and repeat the process. This is from https://www.youtube.com/watch?v=fMZMm_0ZhK4 and I'm not sure if it is really true. But initial range seems to be configurable and we can make it same as the range of short url letters. If all ranges are used, we can allocate new larger ranges and continue.
    - Use a specialized database for managing IDs. However this will become the bottleneck if the QPS becomes high, and it is SPF.
    - To scale the previous solution, we can use multiple servers to generate the unique IDs. For each server, it combines timestamp of the received request, server id and server local counter value and form a unique 64-bit ID(probablility of collsion can be seen as 0, as it is highly unlikely that a large number of requests are sent to the same machine at exactly same time). Since the timestamp takes the MSBs, the generated ID can be seen as sequential. However since the clocks on the distributed servers can be out of sync, so the generated ID may not well reflect the true order. This solution is used by Twitter's ID generator [Snowflake](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake.html). Redis also has similar solution: http://engineering.intenthq.com/2015/03/icicle-distributed-id-generation-with-redis-lua/   
    Ref: http://blog.gainlo.co/index.php/2016/06/07/random-id-generator/?utm_campaign=quora&utm_medium=How+do+you+generate+monotonically+increasing+sequence+numbers+(seqnums)+in+a+large+distributed+system%3F&utm_source=quora
    The downside of it is that the generated ID could be very long, and not consequtive so base conversion would be difficult.
    - The distributed ID generater can produce IDs in another way: say there are 1000 machines, we let machine 1 generate IDs 1, 1001, 2001..., and let machine 2 generate IDs 2, 1002, 2002..., and so on. This should be enough, but may reduce the ID numbers significally if one machine is down.
    Ref: https://www.zhihu.com/question/29270034
    - [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)/GUID. There is still a chance of collision, although it's very low.
* In sumary, when the tiny URL servers receive a long URL to be converted, they should first send a request to the global unique ID generator and get a ID back, and then convert this ID to a base 62 ID, which is the key part of the short URL. We then save the mapping of ID to the long URL in the database. For a 64-bit unique ID, the number of digits in the 62-base ID is around 11. If we want to use less number of digits in the short URL, we can modify the ID generator to return shorter ID, or alow more symbols in the short URL to increase the base. We can just return serverId + counter_on_server, or use range-based id generator like zoo-keeper.

### Custom URL
Just create a new table <Custom_url(key), long_url>. I don't think we need to de-dup since this feature is meant to allow user intentinally create a custom url for a given long url and it is okay to have mutiple different custom url for one long url.

### About recycling short urls
It might be better to generate url_ids/short_urls offline and store them in seperate DBs. Make the data sorted by id, and have a seperate service loading the first ones into the memory. That service can serve as the actual ID sender, instead of the orginal id generater. The good thing about it is that it makes it easier to use recycled short urls. We can add insertion_time and expiration length(and is_deleted) to the <url_id, long_url> table and have an another job periodically check and remove the rows in it. Once a row is removed, put the corresponding url_id/short_url into the short_url DB. 