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


## Services and Storage
* Auth service for registering, logging in and out.
* User service for viewing and editng user profiles.
* Friendship service for managing and storing friends.

### Authentication Service
* User account table.
  | Column     | Type         | Meaning                           |
  |:--------:  |:---------:   |:-----------:                      |
  |user_id     | int          | Primary key, User id              |
  |user_name   |varchar(100)  | Registered user name              |
  |email       |varchar(254)  | Registered user email             |
  |password    |varchar(200)  | hash_algorithm(user_name + salt)  |
  |salt        |varchar(50)   | Nullable                          |
  |password hash algorithm | varchar(50) |                        |
  Salt is used to make it more costly for bulk hack, if the attacker got the info of this table. It is usually explitly stored in the table.
  Hash algorithm column can make it easy to migrate old user info using different hash algorithm to this table, so that the password can be kept. 
  Ref: http://www.vertabelo.com/blog/technical-articles/how-to-store-authentication-data-in-a-database-part-1  
  https://security.stackexchange.com/questions/66989/how-does-a-random-salt-work

  This table can be stored in SQL. Shard it if necessary. Cache of user_name to actual account info is often necessary since the user_name is not primary key and quries like logging in may be slow if this DB is sharded. 

* Use sessions. Session table looks like:

  | Column     | Type      | Meaning                         |
  |:--------:  |:---------:|:-----------:                    |
  |session_key | string    | A unique hash code              |
  |user_id     |Foreign key|Point to the user table          |
  |expire_at   |timestamp  |When to invalidate the session id|

  Can be stored in either a database or a cache.

### User Service
* User profile table is simple to design. May be make it as a different table and not included in the account table is better since it is accessed by a different service. Store in either SQL or NoSQL, shard the table and cache the data if necessary.
