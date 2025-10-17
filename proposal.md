
## Architecture Design Proposal
This section comprises the suggestions for the future development of this project. It proposes a software design to achieve the following architecture drivers:
### 1. Security
#### i. Authentication: Implement JWT with short-lived access token and refresh token
* Use RSA SHA-256 Algorithm for asymmetric sigining where a private key signs tokens and a public key verifies it.
* After the login, the user needs to store the access token and refresh token in cookies.
* The access token will be passed in the headers for all the authenticated APIs. 
* Access token can be short lived (15 minutes) and they need to fetched from a /refresh API once they expire.

#### ii. Secure with TLS Certificates
* To protect the data in transit, we will implement TLS.
* TLS Certificate will protect the data transmitted from man in the middle attack.
* We can create secret TLS file in Kubernetes to be used in the Ingress file.

#### iii. Rate limiter
* DDOS Attack can be avoided with the implementation of Rate limiter with Redis cache based on the user id and the IP address 
* Think of a number of requests that can be sent in a window of a duration. <br />
For example - Only 100 requests can come from the same IP address in 1 minute's window

#### iv. End-to-End Encryption (E2EE) for messages
* The messages are encrypted using a shared secret derived from the recipient's public key and the sender's ephemeral private key.
* In the table, only the encrypted messages will be stored which will be decrypted on the frontend.
* Ephemeral keys will be generated for each message to avoid the decryption of other messages.

#### v. SQL Injection Prevention and Cross site scripting prevention.
* Use a middleware for sanitization of the query, payload, etc coming from the client side.


### 2. Availability
#### i. Multi-region Deployment with failover
* Implement the global load balancer to route the request to server locate in the closest region.
* Latency Reduction: There will be a reduction in latency for the users if the request is being served from the closest server
* High Availability: If one region fails, traffic automatically fails over to the nearest healthy one.

#### ii. Database replicas
* Use PostgreSQL replicas for the read operations to ensure availabilty of the database.
* The database closest to requesting client will be accessed reducing the latency.

#### iii. Connection pooling
* Use PgBouncer for connection management, so that, once the expensive operation of the database connection is established then we reuse the same connection rather than recreating the new connection for every API call.
* The pool provides the connection from its cache or creates a new one.

#### iv. CDN Caching
* For static files, upload them to CDN for faster delivery and upload.
* Avoid uploading them via. the server to avoid unnecessary overhead.

### 3. Reliability and Robustness
#### i. Circuit Breaker Pattern
* If one of the microservice is failing multiple times for a certain duration of time then we are wasting resources of the client microservice as well as serving microservice.
* With this pattern, we will stop calling the serving microservice which will increase it's recovery speed.

#### ii. Blue-Green Deployment Strategy
* Whenever we have to roll out a new version of the application, we will use this deployment strategy to avoid downtime.
* Using this deployment strategy, we will always to be able to serve the requests because the traffic will be switched between blue and green environments depending on it's readiness.

#### iii. Graceful Shutdown and Connection Draining
* During deployment we should implement graceful shutdown to avoid any open database connections or cache connections.

#### iv. Observability and monitoring
* Use Prometheus for metrics collection and Grafana for visualization and alerting.
* If there is a sudden spike in traffic or if there are multiple failures of a service then the Developer/DevOps team will be notified via Slack/Email.     

#### v. Automated testing
* In the CI/CD pipeline's workflow, we should include scripts for the automated testing.

#### vi. Queuing system
* Maintain 2 separate microservices, one for the socket connections and the other one for fetching messages via. API and saving the messages.
* Using Apache Kafka, we will send the message from the socket microservice to the messaging microservice which deal with updating the database.

### 4. Database clarification
1. We are using the UUID as the primary key because it is unique for every table and it won't be repeated unlike BIGINT.
2. We have maintained separate tables for private chats and group chats because it's efficient for complex querying, since the structure will be consistent and none of the columns will have to be stored as null.
3. Members of a group have a many to many relationship with the users table, thus, we are maintaining a `groupMembers` table because one user can be in multiple groups and one group can have multiple users.
4. Based on the updated delivery status of the message, entries will be made either in `privateChatsStatus` table or `groupChatsStatus` table.
5. In order to avoid expensive aggregate count queries, we are maintaining the `MembersCount` in the table.