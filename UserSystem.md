# User System

## Scenario 

### Features
* Register
* Login/out
* View/edit user profiles
* Add/delete friends/followers.

### Traffic
Read QPS ~= 100K, write QPS ~= 100  
Usually read-heavy.

## Services
* Auth service for registering, logging in and out.
* User service for viewing and editng user profiles.
* Friendship service for managing and storing friends.

### Authentication Service
* Use sessions. Session table looks like:

| Column     | Type      | Meaning                         |
|:--------:  |:---------:|:-----------:                    |
|session_key | string    | A unique hash code              |
|user_id     |Foreign key|Point to the user table          |
|expire_at   |timestamp  |When to invalidate the session id|

Can be stored in either a database or a cache.

### User Service
* Usually one MySQL server is enough to store 7 billion people's data. To handle the reads, use a cache like Memcached.
* Other choice? TBD.

### Friendship Service
* One direction friendship is easy to store. 
* Two directions friendship can be represented by two records or one record.
  * If A and B become friends, store A -> B and B -> A. The query for finding the friends of A only needs to check one column. But need to write twice when saving a new friendship. And there could be consistency issue if one of the write fails. We can solve this by using a transaction, or have a cron job to correct the inconsistencies periodically.
  * If A and B become friends, store A -> B only. The query for finding the friends of A needs to check both columns.

## Storage
