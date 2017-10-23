# NewsFeed system 

## Scenario
### Core use cases
* News feed
* Post a tweet
* Timeline
* Follow / Unfollow a user

### Estimation
* Requests sent from the client(analyzed from the use cases):
  - Get news feed: {friend_id, post} <- read(user_id, last_timestamp)
  - Get timeline: {post} <- read(user_id, last_timestamp)
  - Post a tweet: write(user_id, post_info, timestamp)
  - follow/unfollow a user: write(user_id, friend_id)
* Assuming DAU = 150M. Each user uses the app 1 hour every day, which means the concurrent user is 150M / 24 ~ 6M. Say the client side sends request every 1 mintues only when the user is using it, then the read QPS = 6M / 60 = 100K. Peak read QPS ~ 300K. Assuming each user post 6 tweets per day, write QPS = 150M x 6 / 86400 = 10K. 
* Storage: Assuming each tweet has 140 characters and each character need two bytes to store, and meta data takes 30 bytes. Then 150M x (280 + 30)B x 6 = 276GB new data per day.

## Service

### Friendship service
* Follow
* Unfollow

### Tweet service
* Post a tweet
* Newsfeed
* Timeline

### Media service
* Upload image
* Upload video


## Storage
* User table

  | Columns  | Type    |
  | -------- | ------- |
  | id       | Integer |
  | username | varchar |
  | email    | varchar |
  | password | varchar |

  There are around 7 billion people in the world. Say each person takes 40 bytes, then we only need 280GB to store those data. One MySQL server should be enough for that. So we can use SQL for user table. See more in [User System](UserSystem.md)

* Friendship table
  `Select * from friendship_table where from_user_id = user_id`

  | Columns      | Type        |
  | ------------ | ----------- |
  | from_user_id | primary key |
  | to_user_id   | primary key |
  
* Tweet table

  | Columns    | Type        |
  | ---------- | ----------- |
  | id         | Integer     |
  | user_id    | foreign_key |
  | content    | text        |
  | created_at | timestamp   |

  Preferably stored in NoSQL as the query is quite simple and the data is not changed.
  id = user_id + created_at, 64 bits should be enough.

* Store images and videos on file system.


## Solution and Scale

### Post tweet
#### Steps for post a tweet
1. Client asks to add a new tweet record
2. Web server asks tweet service to add a new record

```
postTweet(request, tweet)
	DB.insertTweet(request.User, tweet)
	return success
```

#### Complexity
* Post a tweet
 - Post a tweet: 1 DB write

### Read news feed
#### Steps for news feed
1. Client asks web server for news feed.
2. Web server asks friendship service to get all followees.
3. Web server asks tweet service to get tweets from followees.
4. Web server merges each N tweets from each tweet service and merge them together.
5. Web server returns merged results to client. 

```
// each followee's first 100 tweets, merge with a key way sort

getNewsFeed(request)
	followees = DB.getfollowees(user=request.user)
	newsFeed = empty
	for follow in followees:
		tweets = DB.getTweets(follow.toUser, 100)
		newsFeed.merge(tweets)
	sort(newsFeeds)
	return newsFeed

```

#### Complexity
* Algorithm level: 
 - 100 KlogK ( K is the number of friends), though negligible compared with db time.
* System leveL:
 - Get news feed: N DB reads + K way merge
  + Bottleneck is in N DB reads, although they could be integrated into one big DB query. 

#### Disadvantages
* High latency
 - Need to wait until N DB reads finish

### Improved Design
* Design graph:
  ![Diagram](imgs/newsFeed.svg)
* Tweets table has to be sharded by tweet_id. It cannot shard by user_id since tweets posted by the same user would be stored on the same machine, and if that user happens to be a celebrity, that machine could be a hot spot. 
* Maintain a newsfeed cache to store each user's newsfeed. 
  * Assuming we only store tweet_id instead of the actual object, the storage needed is 300M(number of users) x 500 x 2Bytes = 300GB. So we need to use NoSQL database that supports high read QPS -- MemCached or Redis(or better Aerospike). Shard by user_id, and should not have hot-spot problem since each user only look at his own newsfeed. 
  * Note that write to it is done asynchronously, and write new post to the tweet table will return immediately without waiting for the write to the Newsfeed cache. This means that after user B posts a tweet, there might be some delay before user A sees his post showing up in the newsfeed. But as the write operation happens in the memory(newsfeed cache), this delay should not be much, and should be acceptable.
  * For each user, the list of tweet ids are sorted by created_time, and we could use either LinkedList or LinkedHashMap<Time, Id> for it. 
  * What if the user want to browse much older news feed posts that are not cached? Then the query has to go to the tweet table directly. Facebook actually doesn't support that -- the user can only view his newsfeed posts up to some time, or some number.
* Usually the user can get his timeline from the newsfeed cache. We could also optinally have a timeline cache to store them. This might be a good idea since the timeline cache can store much more data for each user than the newsfeed cache.
* On the client side, user can also cache the loaded newsfeed and timeline data in local files. 
* We could also add push servers to this design. This can make the feed update more real-time, but for celebrity this fan-out-on-write could use a lot of resources since the tweet data is copied. Also this only pushes data to the active users, and when user first login, he has to use pull to get the updates. A better way is to use a hybrid solution -- for celebrities, we just push notifications to the followers to let them pull, and for others, we just push the tweets directly to the followers devices. The newsfeed cache is used for storing celebrity tweets only. This way could significantly reduce the read requests sent from the clients, and also not consuming many network resources. Or we can always send tweet_id instead of tweet_object from the push service, but that may require another query using id to get the tweet data.

### Real-life examples
* Facebook – Pull
* Instagram – Push + Pull
* Twitter – Pull  
Pull model is easier to optimize than push model(See next)

### Hot spot / Thundering herd problem
* Cache (Facebook lease get problem)  
Let one query fail and hold the others until the data are loaded into the cache.
* More details?  
  http://www.cs.utah.edu/~stutsman/cs6963/public/papers/memcached.pdf  

## Additional feature: Follow and unfollow
* Asynchronously executed
 - Follow a user: Merge users' timeline into news feed asynchronously
 - Unfollow a user: Pick out tweets from news feed asynchronously
* Benefits:
 - Fast response to users.
* Disadvantages: 
 - Consistency. After unfollow and refreshing newsfeed, users' info still there. 

## Additional feature: Likes and dislikes
### Schema design

* Tweet table

| Columns     | Type        |
| ----------- | ----------- |
| id          | integer     |
| userId      | foreign key |
| content     | text        |
| createdAt   | timestamp   |
| likeNums    | integer     |
| commentNums | integer     |
| retweetNums | integer     |

* Like table

| Columns   | Type       |
| --------- | ---------- |
| id        | integer    |
| userId    | foreignKey |
| tweetId   | foreignKey |
| createdAt | timestamp  |

### Denormalize
* Select Count in Like Table where tweet id == 1, very costly!
* Denormalize: 
 - Store like_numbers within Tweet Table
 - Need distributed transactions. 
* Might resulting in inconsistency, but not a big problem. 
 - Could keep consistency by using a cron job to sync the data periodically.
* Other ways of storing likes in NoSQL database?

## Related questions
* How to find mutual friends?  
http://www.jiuzhang.com/qa/954/  
I personally think using a graph database is a good idea!
* How to implement pagination?  
http://www.jiuzhang.com/qa/1839/
