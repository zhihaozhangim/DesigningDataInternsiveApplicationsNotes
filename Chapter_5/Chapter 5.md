# Replication

*Replication* means keeping a copy of the same data on multiple machines that are connected via a network. The advantages are:

- To keep data geographically close to your users (and thus reduce latency)
- To allow the system to continue working even if some of its parts have failed (and thus increase availability)
- To scale out the number of machines that can serve read queries (and thus increase read throughput)



All of the difficulty in replication lies in handling *changes* to replicated data. 3 popular algorithms for replicating changes between nodes:

- Single-leader
- Multi-leader
- Leaderless



## Leaders and Followers

Each node that stores a copy of the database is called a *replica*.



##### Synchronous Versus Asynchronous Replication

![image](/Users/zhihzhan/Desktop/DesigningDataInternsiveApplicationsNotes/Chapter_5/images/Screen Shot 2020-11-02 at 7.37.53 PM.png)



The replication to follower 1 is *synchronous*: the leader waits until follower 1 has confirmed that it received the write before reporting success to the user, and before making the write visible to other clients. The replication to follower 2 is *asynchronous*: the leader sends the message, but doesn’t wait for a response from the follower.



The advantage of synchronous replication is that the follower is guaranteed to have an up-to-date copy of the data that is consistent with the leader. If the leader suddenly fails, we can be sure that the data is still available on the follower. The disadvantage is that if the synchronous follower doesn’t respond (because it has crashed, or there is a network fault, or for any other reason), the write cannot be processed. The leader must block all writes and wait until the synchronous replica is available again.



In practice, if you enable synchronous replication on a database, it usually means that *one* of the followers is synchronous, and the others are asynchronous. If the synchronous follower becomes unavailable or slow, one of the asynchronous followers is made synchronous. This guarantees that you have an up-to-date copy of the data on at least two nodes: the leader and one synchronous follower. This configuration is sometimes also called *semi-synchronous*



##### Setting up New Followers

Simply copying data files from one node to another is typically not sufficient: clients are constantly writing to the database, and the data is always in flux, so a standard file copy would see different parts of the database at different points in time. 

The usual process is like this:

1. Take a consistent snapshot of the leader’s database at some point in time—if possible, without taking a lock on the entire database.
2. Copy the snapshot to the new follower node.
3. The follower connects to the leader and requests all the data changes that have happened since the snapshot was taken. This requires that the snapshot is associated with an exact position in the leader’s replication log.
4. When the follower has processed the backlog of data changes since the snapshot, we say it has *caught up*. It can now continue to process data changes from the leader as they happen.



## Handling Node Outages

##### Follower failure: Catch-up recovery

On its local disk, each follower keeps a log of the data changes it has received from the leader. If a follower crashes and is restarted, or if the network between the leader and the follower is temporarily interrupted, the follower can recover quite easily: from its log, it knows the last transaction that was processed before the fault occurred. Thus, the follower can connect to the leader and request all the data changes that occurred during the time when the follower was disconnected. When it has applied these changes, it has caught up to the leader and can continue receiving a stream of data changes as before.



##### Leader failure: Failover

An automatic failover process usually consists of the following steps:

1. *Determining that the leader has failed.* There are many things that could potentially go wrong: crashes, power outages, network issues, and more. There is no foolproof way of detecting what has gone wrong, so most systems simply use a timeout: nodes frequently bounce messages back and forth between each other, and if a node doesn’t respond for some period of time—say, 30 seconds—it is assumed to be dead. 
2. *Choosing a new leader.* This could be done through an election process (where the leader is chosen by a majority of the remaining replicas), or a new leader could be appointed by a previously elected *controller node*. The best candidate for leadership is usually the replica with the most up-to-date data changes from the old leader (to minimize any data loss). 
3. *Reconfiguring the system to use the new leader.* Clients now need to send their write requests to the new leader. 



## Implementation of Replication Logs

##### Statement-based replication

In the simplest case, the leader logs every write request (*statement*) that it executes and sends that statement log to its followers. Although this may sound reasonable, there are various ways in which this approach to replication can break down:

1. Any statement that calls a nondeterministic function, such as NOW() to get the current date and time or RAND() to get a random number, is likely to generate a different value on each replica.
2. If statements use an autoincrementing column, or if they depend on the existing data in the database (e.g., UPDATE ... WHERE *<some condition>*), they must be executed in exactly the same order on each replica, or else they may have a different effect.
3. Statements that have side effects (e.g., triggers, stored procedures, user-defined functions) may result in different side effects occurring on each replica, unless the side effects are absolutely deterministic.

It is possible to work around those issues—for example, the leader can replace any nondeterministic function calls with a fixed return value when the statement is logged so that the followers all get the same value. However, because there are so many edge cases, other replication methods are now generally preferred.



##### Write-ahead log (WAL) shipping

The log is an append-only sequence of bytes containing all writes to the database. We can use the exact same log to build a replica on another node: besides writing the log to disk, the leader also sends it across the network to its followers.

The main disadvantage is that the log describes the data on a very low level: a WAL contains details of which bytes were changed in which disk blocks. This makes replication closely coupled to the storage engine.



##### Logical (row-based) log replication

A logical log for a relational database is usually a sequence of records describing writes to database tables at the granularity of a row. Since a logical log is decoupled from the storage engine internals, it can more easily be kept backward compatible, allowing the leader and the follower to run different versions of the database software, or even different storage engines.



##### Trigger-based replication

A trigger lets you register custom application code that is automatically executed when a data change (write transaction) occurs in a database system. The trigger has the opportunity to log this change into a separate table, from which it can be read by an external process. That external process can then apply any necessary application logic and replicate the data change to another system.



## Problems with Replication Lag

Leader-based replication requires all writes to go through a single node, but read-only queries can go to any replica. For workloads that consist of mostly reads and only a small percentage of writes (a common pattern on the web), there is an attractive option: create many followers, and distribute the read requests across those followers. This removes load from the leader and allows read requests to be served by nearby replicas. However, this approach only realistically works with asynchronous replication—if you tried to synchronously replicate to all followers, a single node failure or network outage would make the entire system unavailable for writing. And the more nodes you have, the likelier it is that one will be down, so a fully synchronous configuration would be very unreliable.

Unfortunately, if an application reads from an *asynchronous* follower, it may see outdated information if the follower has fallen behind. This leads to apparent inconsistencies in the database: if you run the same query on the leader and a follower at the same time, you may get different results, because not all writes have been reflected in the follower. This inconsistency is just a temporary state—if you stop writing to the database and wait a while, the followers will eventually catch up and become consistent with the leader. For that reason, this effect is known as *eventual consistency*.



## Reading Your Own Writes

This is a guarantee that if the user reloads the page, they will always see any updates they submitted themselves. There are various possible techniques. To mention a few:

1. When reading something that the user may have modified, read it from the leader; otherwise, read it from a follower. 
2. For example, you could track the time of the last update and, for one minute after the last update, make all reads from the leader. You could also monitor the replication lag on followers and prevent queries on any follower that is more than one minute behind the leader.

3. The client can remember the timestamp of its most recent write—then the system can ensure that the replica serving any reads for that user reflects updates at least until that timestamp.



Another complication arises when the same user is accessing your service from multiple devices, for example a desktop web browser and a mobile app. In this case you may want to provide *cross-device* read-after-write consistency: if the user enters some information on one device and then views it on another device, they should see the information they just entered. In this case, there are some additional issues to consider:

- Approaches that require remembering the timestamp of the user’s last update become more difficult, because the code running on one device doesn’t know what updates have happened on the other device. This metadata will need to be centralized.
- If your replicas are distributed across different datacenters, there is no guarantee that connections from different devices will be routed to the same datacenter.



## Monitonic Reads

Our second example of an anomaly that can occur when reading from asynchronous followers is that it’s possible for a user to see things *moving backward in time*. *Monotonic reads* is a guarantee that this kind of anomaly does not happen. It’s a lesser guarantee than strong consistency, but a stronger guarantee than eventual consistency. When you read data, you may see an old value; monotonic reads only means that if one user makes several reads in sequence, they will not see time go backward— i.e., they will not read older data after having previously read newer data.

One way of achieving monotonic reads is to make sure that each user always makes their reads from the same replica. For example, the replica can be chosen based on a hash of the user ID, rather than randomly.



## Consistent Prefix Reads

This guarantee says that if a sequence of writes happens in a certain order, then anyone reading those writes will see them appear in the same order. One solution is to make sure that any writes that are causally related to each other are written to the same partition—but in some applications that cannot be done efficiently. 



## Multi-Leader Replication

A natural extension of the leader-based replication model is to allow more than one node to accept writes. In this setup, each leader simultaneously acts as a follower to the other leaders.



### Use Cases for Multi-Leader Replication

##### Multi-datacenter operation

In a multi-leader configuration, you can have a leader in *each* datacenter. Within each datacenter, regular leader–follower replication is used; between datacenters, each datacenter’s leader replicates its changes to the leaders in other datacenters.

![img](/Users/zhihzhan/Desktop/DesigningDataInternsiveApplicationsNotes/Chapter_5/images/Screen Shot 2020-11-04 at 2.19.03 PM.png)



##### Clients with offline operation

For example, consider the calendar apps on your mobile phone, your laptop, and other devices. You need to be able to see your meetings (make read requests) and enter new meetings (make write requests) at any time, regardless of whether your device currently has an internet connection. If you make any changes while you are offline, they need to be synced with a server and your other devices when the device is next online. In this case, every device has a local database that acts as a leader (it accepts write requests), and there is an asynchronous multi-leader replication process (sync) between the replicas of your calendar on all of your devices. The replication lag may be hours or even days, depending on when you have internet access available.



##### Collaborative editing

When one user edits a document, the changes are instantly applied to their local replica (the state of the document in their web browser or client application) and asynchronously replicated to the server and any other users who are editing the same document.



### Handling Write Confilicts

![img](/Users/zhihzhan/Desktop/DesigningDataInternsiveApplicationsNotes/Chapter_5/images/Screen Shot 2020-11-04 at 2.44.41 PM.png) 



##### Synchronous versus asynchronous conflict detection

In a multi-leader setup, both writes are successful, and the conflict is only detected asynchronously at some later point in time. At that time, it may be too late to ask the user to resolve the conflict.



##### Conflict avoidance

The simplest strategy for dealing with conflicts is to avoid them: if the application can ensure that all writes for a particular record go through the same leader, then conflicts cannot occur.



##### Converging toward a consistent state

In a multi-leader configuration, there is no defined ordering of writes, so it’s not clear what the final value should be. The database must resolve the conflict in a *convergent* way, which means that all replicas must arrive at the same final value when all changes have been replicated.

- Give each write a unique ID (e.g., a timestamp, a long random number, a UUID, or a hash of the key and value), pick the write with the highest ID as the *winner*, and throw away the other writes. If a timestamp is used, this technique is known as *last write wins* (LWW).
- Give each replica a unique ID, and let writes that originated at a higher-numbered replica always take precedence over writes that originated at a lower-numbered replica. This approach also implies data loss.
- Somehow merge the values together—e.g., order them alphabetically and then concatenate them.
- Record the conflict in an explicit data structure that preserves all information, and write application code that resolves the conflict at some later time.



##### Custom conflict resolution logic

- On write 

  As soon as the database system detects a conflict in the log of replicated changes, it calls the conflict handler. For example, Bucardo allows you to write a snippet of Perl for this purpose. This handler typically cannot prompt a user—it runs in a background process and it must execute quickly.

- On read

  When a conflict is detected, all the conflicting writes are stored. The next time the data is read, these multiple versions of the data are returned to the application. The application may prompt the user or automatically resolve the conflict, and write the result back to the database. CouchDB works this way, for example.



### Multi-Leader Replication Topologies

A *replication topology* describes the communication paths along which writes are propagated from one node to another.

![img](/Users/zhihzhan/Desktop/DesigningDataInternsiveApplicationsNotes/Chapter_5/images/Screen Shot 2020-11-05 at 10.09.48 AM.png)

In circular and star topologies, a write may need to pass through several nodes before it reaches all replicas. Therefore, nodes need to forward data changes they receive from other nodes. To prevent infinite replication loops, each node is given a unique identifier, and in the replication log, each write is tagged with the identifiers of all the nodes it has passed through.

A problem with circular and star topologies is that if just one node fails, it can interrupt the flow of replication messages between other nodes, causing them to be unable to communicate until the node is fixed. The fault tolerance of a more densely connected topology (such as all-to-all) is better because it allows messages to travel along different paths, avoiding a single point of failure.



On the other hand, all-to-all topologies can have issues too. In particular, some network links may be faster than others (e.g., due to network congestion), with the result that some replication messages may “overtake” others.

![img](/Users/zhihzhan/Desktop/DesigningDataInternsiveApplicationsNotes/Chapter_5/images/Screen Shot 2020-11-05 at 10.19.35 AM.png)



Client A inserts a row into a table on leader 1, and client B updates that row on leader 3. However, leader 2 may receive the writes in a different order: it may first receive the update (which, from its point of view, is an update to a row that does not exist in the database) and only later receive the corresponding insert (which should have preceded the update).



## Leaderless Replication

Some data storage systems take a different approach, abandoning the concept of a leader and allowing any replica to directly accept writes from clients.

The client (user 1234) sends the write to all three replicas in parallel, and the two available replicas accept the write but the unavailable replica misses it. Let’s say that it’s sufficient for two out of three replicas to acknowledge the write: after user 1234 has received two *ok* responses, we consider the write to be successful. The client simply ignores the fact that one of the replicas missed the write.

![img](/Users/zhihzhan/Desktop/DesigningDataInternsiveApplicationsNotes/Chapter_5/images/Screen Shot 2020-11-05 at 10.40.00 AM.png)

Now imagine that the unavailable node comes back online, and clients start reading from it. Any writes that happened while the node was down are missing from that node. Thus, if you read from that node, you may get *stale* (outdated) values as responses. To solve that problem, when a client reads from the database, it doesn’t just send its request to one replica: *read requests are also sent to several nodes in parallel*. The client may get different responses from different nodes; i.e., the up-to-date value from one node and a stale value from another. Version numbers are used to determine which value is newer. 



##### Read repair and anti-entropy

The replication scheme should ensure that eventually all the data is copied to every replica. After an unavailable node comes back online, how does it catch up on the writes that it missed?

- *Read repair*

  When a client makes a read from several nodes in parallel, it can detect any stale responses. For example, in Figure 5-10, user 2345 gets a version 6 value from replica 3 and a version 7 value from replicas 1 and 2. The client sees that replica 3 has a stale value and writes the newer value back to that replica. This approach works well for values that are frequently read.

- Anti-entropy process

  In addition, some datastores have a background process that constantly looks for differences in the data between replicas and copies any missing data from one replica to another. Unlike the replication log in leader-based replication, this *anti-entropy process* does not copy writes in any particular order, and there may be a significant delay before data is copied.

Note that without an anti-entropy process, values that are rarely read may be missing from some replicas and thus have reduced durability, because read repair is only performed when a value is read by the application.



##### Quorums for reading and writing

More generally, if there are *n* replicas, every write must be confirmed by *w* nodes to be considered successful, and we must query at least *r* nodes for each read. (In our example, *n* = 3, *w* = 2, *r* = 2.) As long as *w* + *r* > *n*, we expect to get an up-to-date value when reading, because at least one of the *r* nodes we’re reading from must be up to date. Reads and writes that obey these *r* and *w* values are called *quorum* reads and writes. You can think of *r* and *w* as the minimum number of votes required for the read or write to be valid.



The quorum condition, *w* + *r* > *n*, allows the system to tolerate unavailable nodes as follows:

- If *w* < *n*, we can still process writes if a node is unavailable.
- If *r* < *n*, we can still process reads if a node is unavailable.
- With *n* = 3, *w* = 2, *r* = 2 we can tolerate one unavailable node.
- With *n* = 5, *w* = 3, *r* = 3 we can tolerate two unavailable nodes. This case is illus‐ trated in Figure 5-11.
- Normally, reads and writes are always sent to all *n* replicas in parallel. The parameters *w* and *r* determine how many nodes we wait for—i.e., how many of the *n* nodes need to report success before we consider the read or write to be successful.

![img](/Users/zhihzhan/Desktop/DesigningDataInternsiveApplicationsNotes/Chapter_5/images/Screen Shot 2020-11-05 at 10.55.21 AM.png)



### Limitations of Quorum Consistency

If you have *n* replicas, and you choose *w* and *r* such that *w* + *r* > *n*, you can generally expect every read to return the most recent value written for a key. This is the case because the set of nodes to which you’ve written and the set of nodes from which you’ve read must overlap.

However, even with *w* + *r* > *n*, there are likely to be edge cases where stale values are returned. These depend on the implementation, but possible scenarios include:

- If a sloppy quorum is used, the *w* writes may end up on different nodes than the *r* reads, so there is no longer a guaranteed overlap between the *r* nodes and the *w* nodes
- If two writes occur concurrently, it is not clear which one happened first. In this case, the only safe solution is to merge the concurrent writes. If a winner is picked based on a timestamp (last write wins), writes can be lost due to clock skew.
- If a write happens concurrently with a read, the write may be reflected on only some of the replicas. In this case, it’s undetermined whether the read returns the old or the new value.
- If a write succeeded on some replicas but failed on others (for example because the disks on some nodes are full), and overall succeeded on fewer than *w* replicas, it is not rolled back on the replicas where it succeeded. This means that if a write was reported as failed, subsequent reads may or may not return the value from that write. 
- If a node carrying a new value fails, and its data is restored from a replica carrying an old value, the number of replicas storing the new value may fall below *w*, breaking the quorum condition.



### Sloppy Quorums and Hinted Handoff

Should we accept writes anyway, and write them to some nodes that are reachable but aren’t among the *n* nodes on which the value usually lives?

It is known as a *sloppy quorum*: writes and reads still require *w* and *r* successful responses, but those may include nodes that are not among the designated *n* “home” nodes for a value. By analogy, if you lock yourself out of your house, you may knock on the neighbor’s door and ask whether you may stay on their couch temporarily.

Sloppy quorums are particularly useful for increasing write availability: as long as *any w* nodes are available, the database can accept writes. However, this means that even when *w* + *r* > *n*, you cannot be sure to read the latest value for a key, because the latest value may have been temporarily written to some nodes outside of *n*.

Thus, a sloppy quorum actually isn’t a quorum at all in the traditional sense. It’s only an assurance of durability, namely that the data is stored on *w* nodes somewhere. There is no guarantee that a read of *r* nodes will see it until the hinted handoff has completed.