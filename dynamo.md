# Dynamo

## TLDR

- Highly available key-value store
- Sacrifices consistency
- Main ideas: object versioning, application assisted conflict resolution
- Data is partitioned
- Data is replicated using consistent hashing
- Consistency is facilitated among replicas using a decentralized replica synchronization protocol.
- Failure detection is based on a gossip protocol
- Key decision: allowing applications that use Dynamo to make their own decisions about durability & consistency.


## Introduction

- Reliability and scalability of a system is dependent on how application state is managed
- Amazon’s needs:
  - Applications need to be highly available - shopping carts should be modifiable even when the data centers are on fire
  - They need to treat failure as a normal mode of operation - there are always a small, significant # of server and network components that are failing. So we need to be able to treat failure handling as the normal case without impacting the system’s availability & performance.
- To address these, Amazon built S3, Dynamo, etc. Dynamo is used to manage shopping carts and for the application that maintains session state for an active user.
- This paper
  - Shows how different techniques can be used to create a highly available system.
  - Shows that an eventually consistent storage system can be used in production with demanding applications

## Background on Dynamo

 - Usually, prod systems store their state in relational databases. But for the more common use case of state persistence for say, a user session, this approach doesn't scale. many times, all you want is a key value store. The functionality and fancy querying provided by an RDBMS is expensive and requires skilled personnel to handle it.
- How**ever**
   - Most RDBMSs also choose consistency over availability -- which often cannot be afforded.
   - Most of Amazon’s services do not need relational schema.
- Dynamo is ideal for storing small objects (~1 MB)
- ACID: datastores that provide ACID guarantees typically have low availability. Dynamo is ideal for applications that work with weaker consistency guarantees for high availability in exchange.
- Efficiency: Amazon services have strict latency requirements (99th percentile of distribution). So services must be able to configure Dynamo in a way that meets them.


## Things to Consider when designing dynamo

- Traditionally, commercial systems have made replicas synchronously to provide a strongly consistent data access interface -- at the cost of availability.
 - In Dynamo, designers have traded off consistency for availability. The next challenge is dealing with conflicting data.
- **When should we deal with conflicts?** One option is to keep writes complex and reads simple. In such systems, a write may be rejected if the data store cannot reach all (or a majority) replicas at a given time. However, Dynamo aims to be an “always writable” data store. So the complexity of dealing with conflicts is pushed to the reads, so that writes can always take place.
- **Who should deal with conflicts?** This can be done by the data store (server) or the application using it (client). If resolution is done by the data store, the policies it can use are quite simple (like “last write wins”). On the application, the application has much more context around the data and pick a conflict resolution mechanism that is best suited for its use case. Example, we might want to “merge” two shopping carts instead of overwriting an older version with a later version. However, if the application developers don't want to pick their own policy, they can let the data store pick one (like “last write wins”).

## Principles used in the design of Dynamo

1. **Incremental scalability:** Dynamo should be able to scale out one node storage host at a time, with minimal impact on users of the system (and the system itself)
2. **Symmetry:** every node in Dynamo should have the same set of responsibilities as its peers. There should be no distinguished node(s) with extra responsibilities.
3. **Decentralization:** (an extension of symmetry) the design should favour decentralized p2p techniques over centralized control.
4. **Heterogeneity:** The system needs to be able to exploit the heterogeneity or the variety in the infrastructure it runs on. In other words, the work distribution of Dynamo must be proportional to the individual servers (powerful servers get more load than weaker ones)

## Related Works

### P2P systems

- Instructured p2p networks: Freenet and Gnutella
  - Links between peers were established arbitrarily
  - A search query is usually flooded through the whole network to find as many peers as possible that share the data
- Structured p2p networks
  - They use a globally consistent protocol to ensure that any node can efficiently route a search query to some peer that has the needed data
  - Pastry & Chord have routing mechanisms that ensure that queries can be answered within a bounded # of hops
  - To reduce the latency that is caused by multi hop routing, some p2p systems employ O(1) routing where each peer maintains enough routing information locally so it can route any query to the appropriate peer within a fixed # of hops. Oceanstore & PAST were built on top of these routing overlays.

### Distributed File Systems & Databases

- P2P storage systems typically only support flat namespaces. In contrast, distributed file systems hierarchical namespaces.
- Examples:
  - Systems like Ficus & Coda replicate files for high availability - at the cost of consistency - eventually consistent - system level conflict resolution
  - Farsite - a DFS that does not use a centralized server like NFS does, also does high replication for availability & scalability.
  - GFS - master server for hosting the metadata, actual data is split into chunks and stored in “chunkservers”.
  - Bayou - a distributed relational database that allows disconnected operations - eventually consistent - application level conflict resolution


## Interface

- Two operations: get/put
- get(key)
  - Locates the replicas that contain the assocaiated object and returns either one object or multiple conflicting objects
- put(key, object context)
  - Determines where the replicas of the object should be places based on the key and writes the replicas to disk. The context contains metadata about the object that is opaque to the caller and includes info like the version of the object. The system used the context to verify the validity of the object in the put request.
- Dynamo treats both the key and the object as an opaque array of bytes. It applies an MD5 hash to the key to create a unique 128 bit identifier to determine the storage nodes that are responsible for serving this key

## Partitioning

- Mechanism to dynamically partition data over the set of nodes used: consistent hashing
- The output range of a hash function is treated as a fixed circular space/ring. In other words, the largest hash value wraps around to the smallest.
- Each node is assigned random point in this space which represents its “position” on the ring. Data associated with a key is assigned to a node in these steps:
  - (1) hash the data’s key to yield its position on the ring
  - (2) walk clockwise to find the first node with a position larger than the item’s position
- This way, each node becomes responsible for the region in the ring between it and its predecessor node on the ring.
- The advantage of consistent hashing = the departure or arrival of a node only affects its immediate neighbours and the other nodes are unaffected.
- Challenges with basic consistent hashing - 
  - Randomly assigning each node a position on the ring leads to non uniform data and load dist.
  - The basic algorithm does not take into account the heterogeneity (the fact that nodes have different computational power) of the nodes
- Variant of consistent hashing used in Dynamo:
  - Instead of a node → point mapping, there is a node → points mapping, so dynamo uses the concept of virtual nodes. A virtual node LOOKS like a single node in the system, but each node can be “responsible” for more than one v node.

## Replication

- To achieve high availability & durability,Dynamo replicates its data on multiple hosts. Each data item is replicated at N hosts, where N is a parameter that is configured per-instance.
- Each key k is assigned to a “coordinator node”. Coordinator nodes are responsible for the replication of keys that fall in their range. So the coordinator
  - Stores each key locally
  - Replicates these keys at the N-1 successor nodes in the ring
- Result: each node is responsible for the region of the ring between it and its Nth predecessor.
- The list of nodes that is responsible for a key = **preference list**. We will see later that every node in the system can determine which nodes should be in this list for any key k.
- To account for node failures, the PL contains more than N nodes. With the use of v nodes, it is possible that the first N successor positions for a key may be owned by less than N physical nodes. To address this issue, and ensure that the list includes only distinct physical nodes, the PL for a key is constructed by skipping positions in the ring.

## Data Versioning

- Dynamo: eventually consistent → updates to data are propagated asynchronously
- Example: a put() call may return to its caller before the update has been applied to all replicas, so we could have a scenario where a subsequent get() call on that key returns a value that does not have the latest value applied to it.
- Key point: in the absence of failures, there is a bound on the propagation time. But under failure scenarios (server outages, network partitions), updates may not arrive at all replicas for extended periods of time
- Shopping cart: for example, the app requires that the add_to_cart() call never fails and is never forgotten. If the most recent state of the cart is unavailable and a user makes changes to an older version of the cart, that change is still meaningful and should be preserved. But it should not overwrite the currently unavailable state of the cart. So when a customer wants to add to/delete from a cart (translates to a put() call), those changes are applied to whatever version is available (possibly an older version) and divergent versions are reconciled later.
- **How is this done?** Dynamo treats each the result of each modification to the cart as a new and immutable version of the data. It allows for multiple versions of an object to be present in the system at the same time.
   - Most of the time, new versions don’t conflict with older ones and the system can determine by itself which is the authoritative (most current) version.
   - Sometimes, versions of the data branch off, which results in conflicting versions of an object. In this case the system cannot reconcile them and this task falls to the client. The client performs the reconciliation to “collapse” multiple versions in to one.
- Note that certain types of failure can cause not just 2 but muliple conflicting versions of the object.
- **Vector Clocks**
  - Dynamo uses vector clocks to determine causality between conflicting versions of the same object. A vector clock is a list of (node, counter) pairs. Each version of every object → associated with a vector clock. They help in determining whether two versions of an object are on parallel branches or have a causal ordering. (if parallel → system reconciles, otherwise the client reconciles)
  - If counters on the first object’s clock are less than/equal to those on the second, then the first is an ancestor of the second and can be discarded. Otherwise, the two changes are considered to be in conflict → reconciliation must be done.
  - When a client wishes to update an object, it must specify which version it is updating. So a the client first does a read and then for the update, passes the context it obtained (containing vector clock information). When a client does a read, if the backend has several branches in conflict that it cannot reconcile, it will return all the objects at the leaves with their version information. When this context is then used for a future update (client resolves conflicts and makes an update() request), the conflicting branches are considered to be resolved and the diverging branche are considered to be collapsed or resolved into one.
- Possible issue: vector clocks may grow if many servers coordinate writes to an object. However, in practice, writes are usually handles by one of the top N nodes in the preference list. In case of server failures/network partitions, write requests may be handled by nodes that are not in the top N nodes in the pref list, causing the size of the vector clock to grow. In this case, we can limit the size of the vector clock by specifying some threshold. In theory, this can cause problems if descendant relationships cannot be accurately determined, but in practice, this problem hasn’t surfaced in production (for Amazon).

## Execution of get() and put()

- Any storage node in Dynamo can receive a get() or put() call.
- Clients can pick nodes in 2 ways:
  - (1) route the requests through a load balancer that will select a node based on load information -- the client does not need to link to dynamo specific code.
  - (2) use a partition aware client library that routes requests directly to the appropriate coordinator nodes. -- lower latency because it skips the “forwarding step”
- Node handling a read/write → coordinator. Coordinators are usually the top N nodes in the preference list. (Note: if requests are routed through a load balancer to a node that isn’t in the top N of the preference list, then it forwards the request to the first of the top N)
- To maintain consistency amongst replicas, Dynamo uses a consistency protocol that is similar to those used in quorum systems. The protocol has 2 configurable values:
  - R: min # of nodes that must participate in a read operation
  - W: min # of nodes that must participate in a write operation
- Setting R  + W > N = quorum system. In this mode, the latency of a get or put is determined by the slowest of the R or W replicas. For this reason, R and W are usually less than N (for lower latency)
- put()
  - Upon receiving a put() request for a key, the coordinator generates the vector clock for the new version and writes the new version locally. The coordinator then sends the new version (+ the new vector clock) to the N highest ranked reachable nodes. If at least W-1 nodes respond, then the write is considered to be successful.
- get()
  - The coordinator requests all existing versions of data for that key from the N highest ranked reachable nodes in the preference list for that key and then waits for R responses before returning the result to the client.
  - If the coordinator gathers multiple versions of the data, it returns all the versions it deems to be causally unrelated. The divergent versions are then reconciled by the client and the reconciled version (with all conflicts resolved) then supersedes all previous ones.

## Handling Failures: Hinted Handoff

- If Dynamo used a **traditional quorum** approach, it would be unavailable during server failures and network partitions and would not have very good durability under failure conditions.
- So it uses a **sloppy quorum** approach instead: all read and write operations are performed by the first N healthy nodes from the preference list. Note that this many not always be the first N nodes encountered while walking the consistent hashing ring.
- If a write was sent to node B (because the previous node, node A was down), B would also have a hint in its metadata that suggests that actual node that was intended recipient of the replica (in this case, A). Nodes that receive **hinted replicas** keep them in a separate local database that is polled periodically. When B detects that A has recovered, B will attempt the deliver that replica to A.
