# Location-based service

## Scenario
### Features
* First stage
  * Driver reports locations(like sending heartbeats). The server cannot send requests asking for driver's location as it doesn't know them without drivers reporting to them. Usually driver(or more specifically, the app installed on the driver's phone) reports his location periodically. This is more efficient than letting serve send request and get response back.
  * Rider requests Uber, match a driver with rider
* Second stage
  * Driver denies / accepts a request
  * Driver cancels a matched request 
  * Rider cancels a request
  * Driver picks up a rider and starts a trip
  * Driver drops off a rider and ends a trip
* Third stage
  * Uber Pool
  * Uber Eat

### Traffic
* Assuming 200K are online at the same time, and each driver reports his location every 4 sec.  
  Driver QPS = 200K / 4 = 50K  
  Peak QPS = 50K x 3 = 150K  
  These are writes!
* Rider doesn't have to report his location. And it must be much smaller than Driver QPS.
