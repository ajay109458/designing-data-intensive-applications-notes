
* **Replication** is one of the strategies for distributing data processes across multiple nodes (the other is **partitioning**, the subject of a later chapter).
* The difference between replication and partitioning is that whilst replication has you keep multiple copies of the data in several different places, partioning has you split that one dataset amongst multiple partitions.
* Both strategies are useful, but obviously there are tradeoffs either way.


* Replication strategies fall into three categories:
  * **Single-leader** &mdash; one service is deemed the leader. All other replicas follow.
  * **Multi-leader** &mdash; multiple services are deemed leaders. All other replicas follow.
  * **Client-driver** &mdash; not sure.
  
## Single-leader replication

* The first important choice that needs to be made is whether replication is synchronous or asynchronous.
* Almost all replication is done asynchronously, because synchronous replication introduces unbounded latency to the application.
* User-facing applications generally want to maintain the illusion of being synchronous, even when the underlying infrastructure is not.
* Asynchronous replication introduces a huge range of problems that you have to contend with if you try to maintain this illusion.
* (personal note) GFS is an interesting example. GFS was explicitly designed to be weakly consistent. That unlocked huge system power, and the additional application layer work that was required to deal with the architecture was deemed \"worth it\" because it was on the engineer's hands.


* The precise configuration of concerns with a single-leader replication strategy differs. At a minimum the leader handles writes, and communicates those writes to the follower, and then the followers provide reads.


* If a follower fails, you perform catch-up recovery. This is relatively easy (example, Redis).
* If a leader fails you have to perform a **failover**. This is very hard:
  * If asynchronous replication is used the replica that is elected the leader may be \"missing\" some transaction history which occurred in the leader. Totally ordered consistency goes wrong.
  * You can discard writes in this case, but this introduces additional complexity onto any coordinating services that are also aware of those writes (such as cache invalidation).
  * It is possible for multiple nodes to believe they are the leader (Byzantine generals). This is dangerous as it allows concurrent writes to different parts of the system. E.g. **split brain**.
  * Care must be taken in setting the timeout. Timeouts that are too long will result in application delay for users. Timeouts that are too short will cause unnecessary failovers during high load periods.
* Every solution to these problems requires making trade-offs!


## Replication streams

* How do you implement the replication streams?


* The immediately obvious solution is **statement-based replication**. Every write request results in a follower update.
* This is a conceptually simple architecture and likely the most compact and human-readable one. But it's one that's fallen out of favor due to problems with edge cases.
* Statement-based replication requires pure functions. Non-deterministic functions (`RAND`, `NOW`), auto-incrementation (`UPDATE WHERE`), and side effects (triggers and so on) create difficulties.


* **Write-ahead log shipping** is an alternative where you ship your write-ahead log (if you're a database or something else with a WAL).
* This is nice because it doesn't require any additional work by the service. A WAL already exists.
* This deals with the statement-based replication problems because WALs contain record dumps by design. To update a follower, push that record dump to it.
* The tradeoff is that record dumps are a data storage implementation detail. Data storage may change at any times, and so the logical contents of the WAL may differ between service versions.
* The write-ahead logs of different versions of a service are generally incompatible! But your system will contain many different versions.
* Generally WALs are not designed to be version-portable, so upgrading distributed services using WAL shipping for replication requires application downtime.


* **Logical log replication** is the use of an alternative log format for replication. E.g. replication is handled by its own distinct logs that are only used for that one specific purpose.
* This is more system that you have to maintain, but it's naturally backwards and forward compatible if you design it right (using e.g. a data interchange format like Protobufs) and works well in practice.


* Post-script: you may want partial replication. In that case you generally need to have application-level replication. You can do this in databases, for example, by using triggers and stored procedures to move data that fits your criteria in specific ways.


* Asynchronously replicated systems are naturally **eventually consistent**. No guarantees are made for when \"eventually\" will come to pass.
* **Replication lag** occurs in many different forms. Some specific types of lag can be mitigated (at the cost of performance and complexity), if doing so is desirable for the application (but you can only recover a fully synchronous application by being fully synchronous).
* The book cites three examples of replication lag that are the three biggest concerns for systems.


* The first one: your user expects that data they write they can immediately read. But if they write to a leader and read from a replica, and the replica falls behind the leader, they may not see that data.
* A system that guarantees users can see their own modified data on refresh is one which provides **read-your-write consistency**. In practice, most applications want to provide this!
* You can implement read-your-writes in a bunch of different ways, but the easiest way is to simply have the leader handle read requests for user-owned data.


* Another issue occurs if a user is reading from replica, and then switches to another replica that is further out of sync than the previous one. This creates the phenomenon of going backwards in time, which is bad.
* Applications which guarantee this doesn't happen provide **monotonic reads**.
* An easy way to get monotonic reads is to have the user always read from the same replica (e.g. instead of giving them a random one).


* The third kind of inconsistency is **causal inconsistency**. Data may get written to one replica in one order, and to another in a different order. Users that move replicas will see their data rearranged.
* Globally ordered reads require that the system be synchronous, so we can't recover all of the order. However, what we can do is gaurantee that events that have a causal relationship (e.g. this event happened, which caused this event to happen) *are* mirrored correctly across all nodes.
* This is a weaker gaurantee than fully synchronous behavior because it only covers specific subsequences (user stories) from the data, not the entire sequence as a whole.
* A system that handles this problem is said to provide **consistent prefix reads**.


* It's notable that you can avoid all of these problems if you implement **transactions**. Transactions are a classical database feature that was oft-dropped in the new world because it is too slow and non-scalable for a distributed system to implement. They seem to be making a comeback now, but it's still a wait-and-see.


## Multi-leader replication
* Multi-leader replication is a more complex but theoretically more highly available architecture.
* If there are multiple leaders then if one leader goes down a failover is not necessary (as long as *all* of the leaders don't go down, and the remaining leaders can handle the traffic).
* In addition to the additional durability, there are also various advantages in terms of latency and robustness in the face of network partitions.
* However due to implementation complexity they tend to only be implemented for multi-datacenter operations.
* The fundamental new problem that multi-leader architectures introduce is the fact that concurrent authoritative writes may occur on multiple leaders.
* Thus your application has to implement some kind of **conflict resolution**. This can occur on the backend side, or it may be implemented on the application side. It may even require user intervention or prompting.
* Interestingly, applications which sync data between clients but whose clients are intended to work offline are technically multi-leader, where each leader is the individual application. E.g. calendar applications.

## Client-driven replication
* This is the last of the three main architectures.
* In this case you have \"smart clients\", e.g. clients that are responsible for copying data across replicas (or alternatively, designated nodes that do this on behalf of the clients).
* Client-driven replication were for a long time not a particularly popular strategy because it requires establishing lots of network connections between the client and the replicas. It's what Dynamo used (though it's not what DynamoDB, the hosted product, uses), and that's made it popular. Cassandra does this form example.


* How do you handle data replication without designated leaders?
* Services using client-driven replication do so using two processes.
* A **read-repair** occurs when the client asks for data, gets it, finds that some of the nodes are responding with old data, and forwards a \"hey, update\" to those nodes.
* An **anti-entropy process** constantly or ocassionally scans for inconsistencies in the data nodes, and repair them.
* Clients read using **quorums** of certain sizes, taking the most recently updated data point from amongst the nodes that respond to a request. Quorums offer a tunable consistency-speed tradeoff. Larger quorums are slower, as they are tied down by the speed of the slowest respondent, but more consistent. Smaller quorums are faster, requiring fewer node responses, but less consistent.


* A difficulty that arises with this architecture is what to do is the nodes that are normally responsible for a quorum write are unavailable.
* The strategy is to perform a **sloppy quorum**: write that data to the right amount of still-available nodes.
* Once the nodes that are the true home for that data become available again, perform a **hinted handoff** to those nodes.
* Reader note: the great AWS EC2 node failure outage of 2013 or thereabout occurred because this mechanism floored the network."

