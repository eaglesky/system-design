# Location-based service

## Scenario
### Use Cases
* First stage
  * Driver reports locations(like sending heartbeats). The server cannot send requests asking for driver's location as it doesn't know them without drivers reporting to them. Usually driver(or more specifically, the app installed on the driver's phone) reports his location periodically. This is more efficient than letting serve send request and get response back.
  * Passenger can see the drivers around him in real time on the map.
* Second stage
  * Passenger requests Uber, match a driver with passenger
  * Driver denies / accepts a request
  * Driver cancels a matched request 
  * Passenger cancels a request
  * Driver picks up a passenger and starts a trip
  * Driver drops off a passenger and ends a trip
* Third stage
  * Uber Pool
  * Uber Eat

### Estimation
* The server cannot send requests asking for driver's location as it doesn't know them without drivers reporting to them. Usually driver(or more specifically, the app installed on the driver's phone) reports his location periodically. This is more efficient than letting server send request and get response back. Also the latter do not save any bandwidth or CPU.
* Assuming 200K are online at the same time, and each driver reports his location every 4 sec.  
  Driver QPS = 200K / 4 = 50K  
  Peak QPS = 50K x 3 = 150K  
  Reporting locations is where writes mainly come from, so write QPS is 150K.
* Passenger doesn't have to report his location. And it must be much smaller than Driver QPS. Assuming daily passenger number is similar to the driver's, and each client app updates the drivers' locations every 4 sec, the read QPS is then similar to write QPS -- 150K.
* Storage. (may not need to estimate the number first)  
  If we store all the locations: 200K * 86400 / 4 * 100bytes(length of each record)~ 0.5 T / day.
  If we just store the current locations: 200K * 100bytes = 20MB. 

## Service
* Geo Service. 
* Dispatch Service.

## Storage

