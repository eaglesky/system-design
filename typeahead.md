# Typeahead

## Scenario
* Google suggestion
	- Prefix -> top n hot key words
	- DAU: 500M
	- Search: 6*6*500M = 18b (Every one search for 6 words, each word has 6 characters)
	- QPS = 18b / 86400 ~ 200k
	- Peak QPS = QPS * 2 ~ 400k
* Twitter typeahead

## Initial design
* Query service
	- Each time a user types a character, the entire prefix is sent to query service.
* Data collection service

## Storage
### Query service DB
#### Word count table
* How to query on the db
* Query SQL: Select * from hit_stats where keyword like ${key}% order by hitCount DESC Limit 10
	- Like operation is expensive. It is a range query. 
	- where keyword like 'abc%' is equivalent to where keyword >= 'abc' AND keyword < 'abd'

| keyword | hitCount | 
|---------|----------| 
| Amazon  | 20b      | 
| Apple   | 15b      | 
| Adidas  | 7b       | 
| Airbnb  | 3b       | 

#### Prefix table
* Convert the word count table to a prefix table, stored in databases and their caches. 

| prefix | keywords                     | 
|--------|------------------------------| 
| a      | "amazon","apple"             | 
| am     | "amazon","amc"               | 
| ad     | "adidas","adobe"             | 
| don    | "don't have", "donald trump" | 

Scan the word count table and for each word, check each prefix, and add the word to the set corresponding to each prefix. This process doesn't have to be very fast since it is offline, done by the data collection service periodically. But how to update this table?
I think we might be able to store this prefix table in Redis, so accessing it would be fast and it can persist data. The key is the prefix, and the value could be a min-heap that stores k most popular words with that prefix, and their associated frequencies. Suppose we have a new frequency for "amazon", then check each of its prefixes and see if it already exists in the heap. If so, update the location of "amazon" in the heap, otherwise compare the new frequency with the top element in the heap and insert it if it is larger(remove the top element at the same time). Does it make sense?

### Trie
* Trie ( in memory ) + Serialized Trie ( on disk ). 
	- Trie is must faster than DB because
		+ All in-memory vs DB cache miss

* Store word count at node, but it's slow
	- e.g. TopK. Always need to traverse the entire trie. Exponential complexity, since to get the n most popular words with a certain prefix, all of the nodes under that prefix in the trie need to be traversed.
* Instead, we can store the top n hot key words and their frequencies at each node, search becomes O(len).

| prefix | keywords                     | 
|--------|------------------------------| 
| a      | "amazon","apple"             | 
| am     | "amazon","amc"               | 
| ad     | "adidas","adobe"             | 
| don    | "don't have", "donald trump" | 

* How do we add a new record {abd: 3b} to the trie
	- Insert the record into all nodes along its path in the trie.
	- If a node along the path is already full, then need to loop through all records inside the node and compared with the node to be inserted. We could store the words and their frequencies in a heap like the first approach.

### Data collections service
* How frequently do you aggregate data(more specifically, the log data that records each search word with its timestamp)
	- Real-time not impractical. Read QPS 200K + Write QPS 200K. Will slow down query service.
	- Once per week. Each week data collection service will fetch all the data within the most recent one week and aggregate them. 
* How does data collection service update query service? Offline update and works online.
	- All in-memory trie must have already been serialized(for backup in case the server is down). Read QPS already really high. Do not write to in-memory trie directly.
	- Use another machine. Data collection service updates query service.  Memory has a limit on the rate at which data can be read from or stored into a semiconductor memory by a CPU(a.k.a. memory bandwidth).If we update the trie while we are using it, the read operations will be affected as some operations have to wait due to the limited bandwidth. However I think the CPU is more likely to be the bottleneck than the memory bandwidth, since the update must take quite a lot of CPU cycles. 
	http://www.nic.uoregon.edu/~khuck/ts/acumem-report/manual_html/ch_intro_bw.html  
	That's why we should do the update on another machine --- data collectio service will aggregate the data into word counts table, copy the deserialized data from the other machine, deserialize the trie from the disk, and update the trie using the data in the word counts table. After it is done, we can have a scheduler at the front of the servers and forward the traffic to the other machine with up-to-date trie. The old machine will be used for update instead.  
	Instead of updating the existing trie, why not just creating a new trie from the word counts table?

## Scale
### How to reduce response time
* Cache result
	- Front-end browser cache the results
* Pre-fetch
	- Fetch the latest 1000 results

### What if the trie too large for one machine
* Use consistent hashing to decide which machine a particular string belongs to. 
	- A record can exist only in one machine. Sharding according to char will not distribute the resource evenly. Instead, calculate consistent hashing code 
	- a, am, ama, amax stored in different machines.

### How to reduce the size of log file
* Probablistic logging. 
	- Too slow to calculate and too large amount of data to store. 
	- Log with 1/10,000 probability
		+ Say over the past two weeks "amazon" was searched 1 billion times, with 1/1000 probability we will only log 1 million times. 
		+ For a term that's searched 1000 times, we might end up logging only once or even zero times. 