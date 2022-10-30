## Chat app take-home assignment

You are hired as the first engineer for a brand new chat app company. Your job is to build out
the initial architecture that will power the chat system for the coming months. Here are the main
requirements that the product team is asking for:

1. Users should be able to chat with each other in real-time i.e. messages should be
   delivered as soon as possible
2. Users should be able to browse past chat history
3. Users should be able to see the online/offline presence of other users
   Your task for this assignment is to describe a system architecture that satisfies these
   requirements. You may draw an architecture diagram or write an essay, but clearly outline the
   components of the system, how they interact with each other, and what the costs/tradeoffs of
   your design are. We value our candidates’ time, so please don’t spend more than 1-2 hours on
   this exercise.

## DESIGN

### Functional requirements:

- users use chat clients (mobile and web) to send messages to each others
- users should be able chat in multiple conversations
- users should be able to view conversations that they are in
- users should be able to view/send/receive text messages and attachments in a conversation
- users should be able to see all the history of a conversation
- users should be able to see online status of the other users

### Out of scopes:
* end to end messaging encryption
* typing indicators

### Non Functional requirements:

### Authentication:

JWT -

### CHAT APIs

GET /conversations?limit=uint&cursor=UTC-time

GET /conversations/:id

GET /conversations/:id/messages?limit=uint&cursor=UTC-time

POST /conversations/:id/messages

POST /conversations/:id/file

### Message Notification API

This section considers how to push new messages sent by one user to others

Option1: Using websocket
A websocket is established between the client and server.

- A user send a message to Chat backend
- Chat backend(s) stores the message successfully
- Chat backend(s) sends websocket event(s) containing the new message to other online users

Pros:
 * Allow to decouple the delivery logic from receiving logic (API calls)
 * Allow other types of notification such as typing indicator, update users status.
 * This opens door for asynchorous architecture for scalability. Asynchrous architecture also allows the app to also handle uploading attachment properly. Especially when there's requirement for large file and videos and malware scanning.

Cons:
 * More costly to implement in both backend and frontend apps. 

Option2: Using polling API:
Users in a conversation use short or long polling to periodically fetch new contents. We could also consider server push

- A user send a message to Chat backend
- Chat backend(s) stores the message successfully
- Other users pull the new conversation's messages from the backend.

Pros:
 * simple implementation, suitable for apps that don't forsee high traffic. 

Cons:
 * hard to scale because clients will keep polling the APIs. 
 
**NOTE**: I'd chose to choose to use Websocket as its advantages is outweighted the http polling. The design for other topics will be made based on the use of weboscket. 

### Data models

#### Users:
 * id: int or uuid 
 * name: string
 * avatarUrl: string

indexes:
 * id <-> query user by id
 * name <-> query user by name

#### Conversations
 * id: int or uuid 
 * metadata: string:json // conversation name, avatar

indexes: 
 * id <-> query conversation by id

#### Messages
 * id: int or uuid 
 * attachmentUrl: string | NULL
 * text: string
indexes: 
 * id <-> query message by id

#### Participants 
This data model holds a many to many relationship between conversations and users; a user can belong to multiple conversations and a conversation can have multiple users

 * id: int or uuid 
 * conversationId: int or uuid
 * userId: int or uuid 

indexes:
 * conversationId <-> query all participants/users belong to a conversations
 * userId: int or uuid <-> query all conversations a user is in


### User Status online/offline:
User status should not be tied to a specific conversation. It should be in Chat App level. 

Option 1: 

We could store this information in a permanent storage with an expiration if  a user status is not updated in the last minute. 
 * isOnline: boolean
 * lastSeen: UTC time

The API will determine if a user is online or not by using this logic:
 * if the user's status is `isOnline = false`, the user is offline
 * if the user's status is `isOnline = true` and `expiration` is within `30s`, the user is online
 * if the user's status is `isOnline = true` and `expiration` is more than `30s`, the user is offline

The client app can send status update of the current user every 15 seconds. Status update can be directly stored in the user model or in a separated storage such as Redis (since we status is not a permanent info, Redis is more preferable). Once received, the backend API can request the websocket component to fanout the user status to other users in the same conversation.

Pros:
  * More rebust system, the user status can be extract into a different service. Status update can be determined not only from if the clients connected to chat but also from connecting with main website where the chat app is one of the component. 
Cons:
  * there's cost on implementation


Option 2: Relies on websocket connection
This option implies that the websocket option was chosen to send message notification to users.

In the websocket option,  userId <-> connectionId is stored in a storage (usually Redis), we could determine status of a user by checking if the user currently has a websocket connection.

Pros:
 * simple solution for early release. We could use this solution temporarily while having the next release to implement the first option without breaking change.

Cons:
 * tightly couple with websocket connection


### Architecture:
The design chose Websocket for notifying new messages to clients 


Option 1:


<img width="625" alt="Screen Shot 2022-10-29 at 5 21 31 PM" src="https://user-images.githubusercontent.com/12070823/198852802-097ad967-5a09-42cc-ae6f-eb77f1fef488.png">

In this option, a single service is respobsile for both handling Rest API calls and develivering websocket events. An example of happy path of a message is:

* User A, B, and C are connected a conversation to chat with each others - Websocket server write connectionId <-> userId of the users to Redis
* User A sends a message to the conversation
* Chat API backend stores the mesages into DB
* Chat API query the conversation's participants to get all users (A, B, and C) of the conversation.
* Chat API requests the Websocket server to send a websocket to all the users of the conversation if connected.
* Websocket server look up the connectionIds of by the userIds and send websocket events to the client.


Pros:
 * simpler architecture and codeimplementation compared to option 2 for early release of the chat product.
 * this architecture implies also single server and deployment of the backend. 
 * Faster to deliver messages since the Chat APIS and websocket server are in a single process. 

Cons:
 * Tightly couple between the Chat APIs and deliver logics. In the case we want to support mobile push notification, all the new code will be bundled into a single source code.
 * It's hard to scale. Because the chat APIs and and the websocket servers are deployed into a single server, they cannot scale independently. To handle high load, a vertical scaling is required. It is because the websocket server will maintain the websocket connection with all the clients, if we choose to scale the service horinzontally, the requests from a user may not reach to the server that has the websocket has the websocket connection with that user. 
 

Option 2:


<img width="870" alt="Screen Shot 2022-10-29 at 5 21 43 PM" src="https://user-images.githubusercontent.com/12070823/198852801-77069273-e888-4437-9785-af8fd166ef23.png">\

Similar to option 1, but the Chat API and Websocket servers are different services. The Chat API communicates with the websocket server through a message broker (such as RabbitMQ). 


Pros:
 * decouple the chat API from websocket and other message delivery logic.
 * allow the services to scale horizontally. To fix the problem of determine which replica of the websocket server owns the connection with a particular user, we can use consitent hashing on userId to determistically determine the websocket replica that owns the connection.

Cons:
 * Costly implementation compared to option 1, we need introduce a message broker, the producer and consumer logics, and consistent hashing mechanism to determine the weboscket server. 
 * Require larger team for maintanance and monitoring the message broker.
 * Slowly to deliver messages to other users compared to option 1 - this is not really a con because in high load, async flow could be faster than a synchonous flow. 
 
 
 ## Database 
