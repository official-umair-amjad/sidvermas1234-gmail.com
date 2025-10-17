# Database Design Thought Process

## 1. Decisions and Assumptions made in the designing process
### 1.a. Decisions
#### i. Users and Groups Relationship
A User can belong to multiple groups and a group can have multiple users.
#### ii. Infinite scroll for Messages screen
The latest message from each user and group has to be grouped and displayed in an infinite scroll list
#### iii. Delivery status of messages
The sender of the message should be able to see the list of the users who have read the messages.
#### iv. Seggregation of Groups
The privacy of messages will depend on whether the group is public or private.
#### v. Distinction between messages
There is a distinction between individual messages and group messages.
#### vi. Content type of message
The message can either store text or a url of media file (uploaded to CDN)

### 1.b. Assumptions
#### i. User base
Initially, there will be thousands of users and after a few months there might be millions of users
#### ii. Accessing of messages
The latest last messages sent/received will be accessed frequently.
#### iii. Monitoring and analytics of the network
The delivery status of the messages will be used to analyze the network.

## 2. Main reasons for selecting SQL approach instead of NoSQL
### 2.a. Complex Relationship Requirement
#### i. Fetch the latest last message from  individuals and groups
In order to fetch messages from individuals and groups, we will need complex queries requiring multiple inner joins which is possible in an SQL approach.
```sql
    (
        SELECT pc."Id", pc."Content", pc."Type", pc."Status", pc."SenderUserId", 
            sender."Username" AS "SenderUsername", 'USER' AS "Category",
            NULL AS "GroupId", NULL AS "GroupName", pc."CreatedAt"
        FROM "privateChats" pc
        INNER JOIN "users" sender ON sender."Id" = pc."SenderUserId"
        WHERE pc."ReceiverUserId" = $1
    )
    UNION ALL
    (
        SELECT gc."Id", gc."Content", gc."Type", gc."Status", gc."SenderUserId", 
            sender."Username" AS "SenderUsername", 'GROUP' AS "Category",
            g."Id" AS "GroupId", g."Name" AS "GroupName", gc."CreatedAt"
        FROM "groupChats" gc
        INNER JOIN "groups" g ON g."Id" = gc."GroupId"
        INNER JOIN "users" sender ON sender."Id" = gc."SenderUserId"
        WHERE gc."GroupId" IN (SELECT "GroupId" FROM "groupMembers" WHERE "UserId" = $2)
    )
    ORDER BY "CreatedAt" DESC
```
#### ii. Structured data model
The structure of the message is always fixed and whether it contains a text or media will be determined by the enum column viz., `type`.
#### iii. Atomicity of SQL
Transactions ensure that the messages and delivery status are inserted atomically.
#### iv. Isolation property of SQL
If the same user reads an unread message from two different devices at the same time then only one entry should be inserted for the read message.
#### v. Supports replicas for scalibility
It can offload the read queries to other replicas, thus, a single database won't have to face bottleneck. Read replicas in multiple geographical regions can decrease the latency.
#### vi. Connection pooling
Establishing the connection with the database can be expensive, pooling reuses the existing connection  improving the performance. 

### 2.b. Weaknesses of this approach
#### i. Write loads
Millions of write operations for saving the messages could lead to bottleneck     
#### Mitigation Strategies:
a) Implement connection pooling to handle thousands of concurrent requests
b) Create replicas to offload the read operations to multiple replicas <br/>
c) Partition the messages by dates

#### ii. Inner joins affecting the performance of query
If we try to fetch the last latest message of multiple users and groups then it can deteriorate the performance.
#### Mitigation Strategies:
We can insert the compute intensive query's result in a `Materialized View` table and fetch it from there.

### iii. Search functionality
Searching messages based on text in SQL can be inefficient.
#### Mitigation Strategies:
We can use ElasticSearch to search the messages based on text efficiently.

### iv. Inefficient aggregate queries
If we have to fetch the count of users who have read the message then running a count aggregate function would be expensive.
#### Mitigation Strategies:
We can store the `ReadCount INTEGER` column in the message tables to fetch the ReadCount and it needs to be incremented if someone reads the message.
