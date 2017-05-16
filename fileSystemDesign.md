# File system

## Scenario
* Write a file(including rename and delete).
* Read a file
* Support file larger than 1000T.
* Use multiple machines to store these files
* No need to support file search for now, as it is usually not a necessary feature.

## Service
* Peer-to-peer
* Master-slave

## Storage
### How to save a file in one machine
* Metadata
	- FileInfo
		+ Name = dengchao.mp4
		+ CreatedTime = 201505031232
		+ Size = 2044323
	- Index
		+ Block 11 -> diskOffset1
		+ Block 12 -> diskOffset2
		+ Block 13 -> diskOffset3
* Block
	- 1 block = 1024 Byte
	- Advantages
		+ Error checking
		+ Fragmenting the data for storage
* About the size of disk block(sector).
  - The disk has to be segmented into many sectors. Otherwise one read or write will operate on the whole track which is very costly. 
  - Drawbacks when the sector size is too small: 
    + Each sector is sensive to media defects and requires a long error correction section. 
    + Other sections in each sector could take up much space when adding them up.
  - Drawbacks when the sector size is too large:
  	+ Read-modify-write is costly for small files. 
  - So choosing the sector size is a trade-off. It used to be 512 bytes for a long time and now has increased to 4KB. 
  - References:
  	+ https://www.mirazon.com/understanding-hard-disk-sector-size-part-1-the-change/
  	+ http://www.seagate.com/tech-insights/advanced-format-4k-sector-hard-drives-master-ti/


### How to save a much larger file in one machine 
* Change chunk size
	- 1 chunk = 64M = 64 * 1024K. Usually multiple of disk sector size.
	- Advantages
		+ Reduce size of metadata -- ones used by meta data(inode) to point to used data blocks, and also the ones pointing to the free data blocks.
		+ If the chunk size is too small, when writing data to a file, it needs to allocate a large number of disk blocks to the file one by one(because free disk blocks are discontiguous), which needs to send many write request to the disk and can be quite costly.
	- Disadvantages
		+ Waste space for small files. If we allow multiple files sharing the same chunk, it could be much harder to manage it and increase the complexity of the file system.

## Scale
### Architecture style
* Peer 2 Peer (BitComet, Cassandra)
	- Advantage: No single point of failure
	- Disadvantage: Multiple machines need to negotiate with each other
* Master slave
	- Advantage: Simple design. Easy to keep data consistent
	- Disadvantage: Master is a single point of failure
* Final decision
	- Master + slave
	- Restart the single master

### How to save an extra large file on several machines
* One master + many chunk servers

#### Move chunk offset from master to slaves
* Master don't record the disk offset of a chunk
	- Advantage: Reduce the size of metadata in master; Reduce the traffic between master and chunk server

### Write process
1. The client divides the file into chunks. Create a chunk index for each chunk
2. Send (FileName, chunk index) to master and master replies with assigned chunk servers
3. The client transfer data with the assigned chunk server.

### Do not support modification

### Read process
1. The client sends (FileName) to master and receives a chunk list (chunk index, chunk server) from the master 
2. The client connects with different server for reading files

### Master task
* Store metadata for different files
* Store Map (file name + chunk index -> chunk server)
	- Find corresponding server when reading in data
	- Write to more available chunk server

### Failure and recovery
#### Single master
* Double master (Apache Hadoop Goes Realtime at Facebook)
* Multi master (Paxos algorithm)

#### What if a chunk is broken
* Check sum 4bytes = 32 bit
* Each chunk has a checksum 
* Write checksum when writing out a chunk
* Check checsum when reading in a chunk

#### Avoid loss of data when chunk server is down
* Replica: 3 copies
	- Two copies in the same data center but on different racks
	- Third copy in a different data center
* How to choose chunk servers
	- Find servers which are not busy
	- Find servers with lots of available disk space

#### How to recover when a chunk is broken
* Ask master for help

#### How to find whether a chunk server is down
* Heart beat message

#### How to solve client bottleneck
* Client only writes to a leader chunk server. The leader chunk server is responsible for communicating with other chunk servers. 
* How to select leading slaves

#### How to solve chunk server failure
* Ask the client to retry