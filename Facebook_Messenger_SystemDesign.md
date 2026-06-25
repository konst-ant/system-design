# Designing Facebook

-------------------------------------------


## System definition

Facebook is a messenger application where users can send text-based instant messages to each other.  
Users can chat with both mobile phone & web site


## Requirements and Goals:

### Primary Functional Requirements
- users should be able to chat with text-base messages one on one
- user messages has to be saved persistently
- messenger should be keeping online/offline status of all users

### Non-Functional Requirements
- users should have real-time chat experience, with minimal latencies during user interaction
- our service should provide strict consistency, with user dialogs being synchronized across all user devices
- availability of system should be respected, however consistency should hit in interest of availability - if some messages don't show up for a short time, it should be fine

### Extended Functional Requirements
- our service should support group chat, where group chart participants write each on group
- system should support push notification on new messages for offline users

##  Capacity Estimation and Constraints

- *500M DAU* assumed

- We will suppose on the average: message is 100 bytes (plus metainformation - we will store sender and receiver IDs - 8 bytes in summary), and each active user generates *40 new messages per day*

- *) 500M * 40 * 108 = 2Tb - storage required per day

- *) 2T * 365 * 5 = 3.5Pb - *5 year period storage volume*

- Bandwidth estimation would be

  2TB / 86400 = 25 MB/s = 100 Mbps - ingress bandwidth
  25 MB/s = 100 Mbps - egress bandwidth

### API 



### High-Level Design

  - The basic flow constitutes:
  
  	User A - user B interaction through the server, who is in responsibility of relaying message and parallel with that persisting it into DB.  
      
  	At that, upon sent of message, A gets an ACK from server; upon receipt of message, B get an ACK from server
  
	<br>
				 
  - Consider a simple case of 1 server instance (for clarity one flow scenario, and then we will extrapolate to heavy load design):
    - receive incoming message at server
    - deliver outgoing message
        
    To get messages from server user has 2 options:
        
      **pull model**
            
      With the pull model user client need to periodically poll server for new messages. In majority of times polling this will return back empty result. This will induce severe network, resource and performance issues, and is not a good solution.
              	
      **push model**  
					
      With push model server communicates back to the client when the message gets available. There are several options how this can be implemented, e.g. using *Websockets*, or the other way using *long polling* when on client request server holds the request open rather than returning empty result, and then server will reply back once new message got available for that user, yet another approach is *SSE (server sent events)*, which also assumes permanently open connection to the client and messages being pushed back by server side. With SSE client will need to re-cock server request upon receipt of new delayed message, and same on timeout disconnection
					
      So, selecting effective push model, it will work so that chat server instance will maintain open connections with active users. 
  
  	<br>
        
  - Let's calculate how many chat server instances will be required then. We will assume 1 server instance can have 50K connection pool. And we also assume to provide online connect for the full audience of DAU:
					
    *) 500M DAU / 50K = 10K - chat server instances required
					
      Now to route user message to the right server we will need a Load Balancer in front to store all active users against their chat server instance. LB will have to dynamically maintain this mapping. On chat server side on the contrary we will adopt internal Hashtable mapping UserID into Connection object (handle). So that on incoming message we fast extract connection from our Hashtable and go ahead with the flow.
					
      So basically we don't keep implicit information about conversations A <-> B, each chat server store only connection to corresponding user. Information about whom user A talk to is stored in message, i.e. each message has sender A and receiver B user IDs.
				
  	This are all aspects regarding application layer processing.

	<br>
			
  - Message delivery for offline user: 
                    
  	We will adopt the general idea, that incoming message sent to offline user will be accepted and stored in DB by chat server, to be delivered once the recipient user returns online. That algorithm can be easily implemented with Hbase storage (as described in store section below) because it will translate in a single search-by-key request in the message store.

	<br>
					
  - Store / retrieve message from DB
			
    For the storage we obviously can not use SQL or NoSQL system such as MongoDB, for one atomic action in our case is saving/retrieving a very short piece of data (message), while the amount of that requests is huge: 500M * 40 / 86400 = 20B / 86400 = 250K rps - this will be the load on DB, so unacceptable to scale up
			
    The suitable solution is to leverage a column-based distributed storage with memory-base fast cache, working with the following technique: each record store key as chat ID (e.g. pair UserA_ID, UserB_ID) against the series of columns (messages) ordered by sequence number or chronologically, and these are stored in memory being popped out to disk on buffer fill-up. This solution is adopted by Hbase, which will deliver a great boost in performance of a back storage with sequential range scans fitting well with the requirement of fast retrieving of user conversation from the storage. Another example of this type of storage is Google's BigTable.
			
    Also note that HBase supports short small updates and searches across the columns
			
    Clients will be fetching messages to their side with a pagination technique using corresponding view window.

	<br>
			
  - Track online status of user, notifying it's peers about these status changes
		
    As we have connection objects cached within fleet of chat server instances, we can effectively extract user status info from there. As we don't store explicitly info about A - B communication (if they chat or no), to update user status cleanly we will need to broadcast each connection status change across all existing connections which is performance crashing operation. We would like to avoid that and will advocate the pull approach, using this scenarios:
        - when user logs in, his client sends a request to know the status of his peers from contact list. Chat server instance delegates status lookup to respective other instances, so we get "distributed in time" operation not overloading the server.
        - even if the user went offline we can show stale info for a while, and we can implement periodic "heart beat" fictitious message from A to B by client, which will just report the status of that peer
        - if user writes a message to a peer who happen to switched offline, we can right away update his status in the client app, and same holds for new party whom our user starts chatting


## Database schema

We use a wide-column storage (HBase / BigTable-like) with log-structured storage model.

Instead of traditional relational schema, data is organized as sorted key-value records.

## Data Sharding

Sharding is handled natively by the underlying wide-column storage system (HBase / BigTable), which partitions data by row key ranges.

Since our row key is **conversation_id**, data is distributed across regions based on conversation_id ranges.

This ensures:
- even distribution of load (large number of conversations)
- horizontal scalability
- efficient range scans within a single conversation (data locality), as well as data write (append)


## High-availability

At application layer we support HA by a redundant fleet of chat servers

With Load Balancing through the fleet of chat servers, our online service is going to be inherently highly available. When any instance goes down, we need client application to support instant reconnect, so the downtime to be reduced to a minimum


## Performance

Cache

Caching is used to reduce read latency and offload the storage layer for frequently accessed data.

We introduce an in-memory cache layer (e.g. Redis-like distributed cache) in front of the message storage.

Caching strategy:

- **Recent messages cache (per conversation_id):**
  - store last N messages of active conversations
  - serves most common access pattern (open chat → load recent history)

- **Hot conversations:**
  - conversations with high read/write activity are kept in cache
  - reduces repeated range scans on storage layer

- **Write-through / write-back strategy:**
  - messages are written to storage first (source of truth)
  - cache is updated asynchronously or in parallel

- **Cache eviction:**
  - LRU or TTL-based eviction for inactive conversations

Cache keys are aligned with storage access pattern:
- key: conversation_id
- value: list of recent messages (ordered by message_id)

This ensures that:
- reads are fast (served from memory)
- storage layer (HBase) is used mainly for long history and durability

## Consistency

For cross-device consistent synchronization of the message list, we need to apply sequential counting of messages, so that despite of client timing experiences we can uniformly order messages on all devices.
The other means to support consistency, to hit the goal of no message is lost, we should apply data redundancy with a sort of replication mechanism.


## Extended functional requirements

- Group chats  
  For the group chats we can have list of participants saved in DB with a key GroupChatId. Separate chat server will be responsible for group chats, having it's logic. When a message comes addressed to the group chat, the server will find char server instances for corresponding group users and forward incoming message to all of them.

- Push notifications  
  This functionality can be implemented with a use of dedicated module (component) - push server, which will be delegated to send push notification towards offline users when a message is received for that user.



## Addition considerations

System may support data compression to reduce storage requirements