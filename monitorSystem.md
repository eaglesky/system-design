# Rate limiter

## Scenario
### Use cases
* Make sure requests having same ip/user_id/email not being processed more than a certain times in the past x seconds/minutes/hours. If the service has received requests more than the threshold in the past specified time, reject the requests with same ip/user_id/email. 
* No need to have too fine granularity

### Estimation
* APIs.
	- Log the data for each request, which is essentially a write to the backend storage: write(event_name, event_value, timestamp)
	- Read the current count for each request, and then compare it with the threshold in the memory: count <- read(event_name, event_value, timestamp, time_limit)
* Say the QPS is x. We need to log each request, so the write QPS to the backend storage is also x. For each request we need to check against the threshold, so the read QPS to the backend storage is also x.
* Read and write heavy.

## Storage
* Need to log who did what at when
	- Event + feature + timestamp as memcached key
		+ event = url_shorten
		+ feature = 192.168.0.1
		+ timestamp
* Memcached
	- Fast for read and write.
	- Large amount of data since there are many ips/users/emails, need distribution.
	- Takes care of data eviction.
	- Do not need persistence. We can recover it quickly if any machine is down, and rebuild the data of past x seconds/minutes/hours quickly. We could have cron jobs taking snapshots perodically to implement persistence that helps recovering data more quickly. -- more on MemCached? (https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf)

## Initial solution
* Say we want to limit the number within the past 1 minute. We can break 1 minute into 60 buckets, each representing 1 second. We record the counts in each second and add them up to find the total count, and then compare it with the threshold. We could also break it into other numbers of buckets, but less granularity could cause more error. Say we break it into only two parts. We would record the count for each 30 secs like: [00:00 -> 50, 00:30 -> 40, 01:00 -> 12, 01:30 -> 33]. Say the threshold for last 1 minute is 100. If another request comes in at 01:02, we would check the sum of 00:30 bucket and 01:00 bucket and it is 52, which is below the threshold. However if the 50 occurrences in the 00:00 bucket happened after 00:02, then the actual number of occurences during the last 1 minute is actually 102, which is above the threshold. Even if we take into account the first bucket, there are still exceptions that reflect similar errors. The root cause is: we add the bucket numbers up to estimate the last 1 minute count, but the actual time range represented by the bucket numbers are not equal to 1 minute, and the difference would become larger if the granularity is coarser. 
* In Memcached the key is Event + feature + timestamp and the value is the count. Use memcached.increment(key, ttl=120s) to increment corresponding bucket, set invalid after 120s. We should use ttl larger than 60s because we will do the get later(see below) and we could get an expired entry if the ttl = 60s.
* Check whether over the limits in the last minute
	```
		// Check the visiting sum in the last minute
		// This part can be parallelized if using distributed Memcached
		for t in 0~59 do
			key = event + feature + (current_timestamp - t)
			sum += memcached.get(key, default=0)
	```
	The actual code is similar to LinkedHashMap solution in [Leetcode] Design Hit Counter.

## Improved solution
* Problem: If we want to find past 1 day number, just use the previous map would require 86400 Memeched get operations, that's too many. How to make it faster?
	- Naive solution : make the bucket coarser by using 1 hour for each bucket. But the error would be larger in this way.
	- Better solution: use multi-level buckets. Besides map of timestamp_sec -> count, we also maintain maps of timestamp_min -> count and timestamp_hour -> count. If a request comes in at 23:30:33, we ideally want to find the total count during (23:30:33 previous day, 23:30:33]. We can do the following:
		+ Look up the hours buckets: [00:00:00, 01:00:00, 02:00:00 ... 23:00:00], add the corresponding counts up. 24 gets. Note that the last bucket contains the count from 23:00:00 to now(23:30:33). 
		+ Look up the minutes buckets: previous day[23:31:00, 23:32:00, ..., 23:59:00]. 29 gets.
		+ Look up the seconds buckets: previous day[23:30:34, 23:30:35, ..., 23:30:59]. 26 gets.
		+ Total number of gets = 24 + 29 + 26 = 79. Acceptable!


# Data log
## Scenario
* Design a service that records the requests number and displays them as a curve graph for any time range specified by the user.
* Typically a user always asks for a certain ts to current.
* A lot of writes and few reads. Assuming QPS of requests processed by webservers is 2K here.

## Storage
* Very large amount of data, structure is simple (key -> value), can be stored using either file system or NoSQL. Graphite actually uses the former.
* If we use NoSQL, the key should be metric_name + timestamp. Value should be the count. Typically I think the granularity can be 1 second. If we use file system, we can create a folder for each metric, and each file in it uses timestamp as the name. File content can be just the count.
* Note that the write QPS is actually way smaller than 2K. Since the client side(webservers) typcially aggregates the gathered data locally(using in-memory hashmap) and sends them to the data log server periodically(let's say 15s here), the actual write QPS to the server is number_web_servers / 15. After webservers push the data out, the hashmap is cleared.
* After the number of keys for a certain metric exceeds a certain threshold, an aggration job will be triggered on the data log server. If we don't check a very granular time range in the past, we can aggregate the past data into per minutes or per hour. This can be done by a cron job running every week/month/year.

## Scaling
* How to shard?
	- We can shard by metric name. For a certain metric, if we store each second's data for a year, the total size is 365 x 24 x 60 x 60 x (8B + 8B) = 504MB. The total size for the last 10 years is 5GB. One machine is definitely enough for at least one metric. If we have more metrics, we can add machines. If we use consistent hashing moving data shouldn't be too hard.

## Example
* Graphite(Read more?)