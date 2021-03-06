
# Transactions

* Transactions provide guarantees about the behavior of data that are fundamental to the old SQL style of operation.
* Transactions were the initial casualty of the NoSQL movement, though they are starting to make a bit of a comeback.
* Not all applications need transactions. Not all applications want transactions. And not all transactions are the same.

##  ACID

* **ACID**, short for atomic-consistent-isolated-durable, is the oft-cited headliner property of a SQL data store.
* Operations are **atomic** if they either succeed completely or fail completely.
* Consistency is meaningless, and was added to pad out the term and make the acronym work.
* Operations are **isolated** if they cannot see the results of partially applied work by other operations. In other words, if two operations happen concurrently, any one will see the state either before or after the changes precipitated by its peer, but never in the middle.
* A database is **durable** if it doesn't lose data. In practice nothing is really durable, but there are different levels of guarantees.

## Single versus multi object operations
* An operation is **single-object** if it only touches one record in the database (one row, one document, one value; whatever the smallest unit of thing is). It is **multi-object** if it touches multiple records.
* Basically all stores implement single-object operation isolation and atomicity. Not doing so would allow for partial updates to records in cases of failures (e.g. half-inserting an updated JSON blob right before a crash), which is Very Bad.
* Isolated and atomic multi-object operations (\"strong transactions\") are more difficult to implement because they require maintaining groups of operations, they get in the way of high availability, and they are hard to make work across partitions.
* So even though distributed data stores that abandon transactions may offer operations like e.g. multi-put, they allow these operations to result in partial application in the face of failures.
* These services work on a \"best effort\" basis. If an error occurs it is bubbled up to the application, and the application layer must decide how to proceed.
* Can you get away without multi-object operations? Maybe, but only if your service is a very simple one.
* All of the things that follow are **race conditions**.


## Weak isolation levels
* The strongest possible isolation guarantee is **serializable isolation**: transactions that run concurrently on the same data are guaranteed to perform the same as they would were they to run serially.
* However serializable isolation is costly. Systems skimp on it by offering weaker forms of isolation.
* As a result, race conditions and failure modes abound. Concurrent failures are really, really hard to debug, because they require lucky timings in your system to occur.
* Let's treat some isolation levels and some concurrency bugs and see what we get.


* The weakest isolation guarantee is **read committed**. This isolation level prevents dirty reads (reads of data that is in-flight) and dirty writes (writes over data that is in-flight).
* Lack of dirty read safety would allow you to read values that then get rolled back. Lack of dirty write safety would allow you to write values that read to something else in-flight (so e.g. the wrong person could get an invoice for a product that they didn't actually get to buy).
* How to implement read committed? 
* Hold a row-level lock on the record you are writing to.
* You could do the same with a read lock. However, there is a lower-impact way. Hold the old value in memory, and issue that value in response to reads, until the transaction is finalized.

* If a user performs a multi-object write transaction that *they* believe to be atomic (say, transfering money between two accounts), then performs a read in between the transaction, what they see may seem anomolous (say, one account was deducted but the other wasn't credited).
* **Snapshot isolation** addresses this. In this scheme, reads that occur in the middle of a transaction read their data from the version of the data (the *snapshot*) that preceded the start of the transaction.
* This makes it so that multi-object operations *look* atomic to the end user (assuming they succeed).
* Snapshot isolation is implemented using write locks and extended read value-holding (sometimes called \"multiversion\").


* Note: repeatable read is used in the SQL standard, but is vague. Implementations like to have a reapatable read level, but it's meaningless and implementation-specific.


* Next possible issue: **lost updates**. Concurrent transactions that encapsulate read-modify-write operations will behave poorly on collision. A simple example is a counter that gets updated twice, but only goes up by one. The earlier write operation is said to be lost.
* There are a wealth of ways to address this problem that live in the wild:
  * Atomic update operation (e.g. `UPDATE` keyword).
  * Transaction-wide write locking. Expensive!
  * Automatically detecting lost updates at the database level, and bubbling this back up to the application.
  * Atomic compare-and-set (e.g. `UPDATE ... SET ... WHERE foo = 'expected_old_value'`).
  * Delayed application-based conflict resolution. Last resort, and only truly necessary for multi-master architectures.


* Next possible issue: **write skew**.
* As with lost updates, two transactions perform a read-modify-write, but now they modify two *different* objects based on the value they read.
* Example in the book: two doctors concurrently withdraw from being on-call, when business logic dictates that at least one must always be on call.
* How do you mitigate?
* This occurs across multiple objects, so atomic operations do not help.
* Automatic detection at the snapshot isolation level and without serializability would require making $n \\times (n - 1)$ consistency checks on every write, where $n$ is the number of concurrent write-carrying transactions in flight. This is way too high a performance penalty.
* Only transaction-wide record locking works. So you have to make this transaction explicitly serialized, using e.g. a `FOR UPDATE` keyword.


* Next possible grade of issue: **phantom write skew**.
* A `FOR UPDATE` will only stop a write skew if the constraint exists in the database. If the constaint is *the lack of* something in the database, we are stuck, because there is nothing to lock. Example of where this might come up: two users registering the same username.
* You can theoretically insert a lock on a phantom record, and then stop the second transaction by noting the presence of the lock. This is known as **materializing conflicts**. 
* This is ugly because it messes with the application data model, however. Has limited support.
* If this issue cannot be mitigated some other way, just give up and go serialized.


## Serialization implementation strategies
* Suppose you decide on full serializability. How do you implement it? What is so onorous that so many data stores choose to drop it?
* There are three ways to implement serializability currently in the market.


* The first and most literal way is to run transactions on a single CPU in a single thread. This is **actual serialization**.
* This only became a real possible recently, with the speed-ups of CPUs and the increasing size of RAM. The bottleneck is obviously really low here. But it's looking tenable. Systems that use literal serialization are of a post-2007 vintage.
* However, this requires writing application code in a very different way.
* The usual way of writing queries is to incrementally update values as the user performs actions. The actions are performed using application code, so the requisite read-update-write loops require constant back-and-forth with the database. This is known as \"interactive querying\".
* Truly serial databases only work if you use **stored procedures** instead.
* Stored procedures are database-layer operations that implement arbitrary code on incoming data.
* Databases classically offered their own stored procedure languages, which were trash. Recent entries in the field embed scripting languages like Clojure or Lua (the latter being what Redis uses) instead, making this much more tenable.
* When you lean on stored procedures over application code, you need to design your applications to minimize reads and writes, and to let the logic be performed on the database level (via the stored procedure), not in the application layer.
* This is very possible, but it is very different...
* Partitioning the dataset whilst using the architecture allows you to scale the number of transactions you can perform with the number of partitions you have.
* However, this is only true if your code only hits one partition. Multi-partition code is *order of magnitudes slower*, as it requires network moves!


* **Two-phase locking** is the oldest technique, and for a long time the only one.
* Two-phase locks are very strong locks, strong enough to provide true serialization. A basic two phase lock performs as follows:
  * Transactions that read acquire shared-mode locks on touched records.
  * Transactions that write acquire, and transactions that want to write after reading update to, exclusive locks on touched records.
  * Transactions hold the highest grade locks they have for as long as the transaction is in play.
* In snapshot isolation, reads do not block reads or writes and writes do not block reads. In two-phase locking, reads do not block reads or writes, but writes block everything.
* Because so many locks occur, it's much easier than in snapshot isolation to arrive at a **deadlock**. Deadlocks occur when a transaction holds a lock on a record that another transaction needs, and *that* record holds a lock that the other transaction needs. The database has to automatically fail one of the transactions, roll it back, and prompt the application for a retry.
* Why is 2PL bad? Because it has very bad performance implications. Long-blocking writes are a problem. Deadlocks are a disaster. Generally 2PL architectures have very bad performance at high percentiles; this is a main reason why \"want-to-be sleek\" systems like Amazon have moved away from them.
* There's one more implementation detail to consider: how 2PL addresses phantom skew.
* You can evade phantom skew by using **predicate locks**. These lock on all data in the database that matches a certain condition, even data that doesn't exist yet.
* Predicate locks are expensive because they involve a lot of compare operations, one for each concurrent write. In practice most databases use **index-range locks** instead, which simplify the predicate to an index of values of some kind instead of a complex condition.
* This lock covers a strict superset of the records covered by the predicate lock, so it causes more queries to have to wait for lock releases. But, it spares CPU cycles, which is currently worth it.


* The third technique is the newest. It is **serializable snapshot isolation**.
* True to its name it is based on snapshot isolation, but adds an additional algorithmic layer that makes it serialized and isolated.
* SSI is an **optimistic concurrency control** technique. **Two-phase locking** is a **pessimistic concurrency control** technique. SSI works by allowing potentially conflicting operations to go through, then making sure that nothing bad happens; 2PL works by preventing potentially conflicting operations from occurring at all. 
* An optimistic algorithm has better performance when there is low record competition and when overall load is low. It has poorer performance when lock competition is high and overall load is high.
* It beats 2PL for certain workloads!
* SSI detects, at commit time (e.g. at the end of the transaction), whether or not any of the operations (reads or writes) that the transaction is performing are based on outdated premises (values that have changed since the beginning of the transaction). If not, the transaction goes through. If yes, then the transaction fails and a retry is prompted.
* For read-heavy workloads, SSI is a real winner!"
